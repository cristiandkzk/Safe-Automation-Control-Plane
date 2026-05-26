# Protocolo: API Integration

> Parte del [Safe Automation Control Plane](./README.md).
> Relacionado con: [AI Routing Decision Engine](./protocol-decision-engine.md) · [AI Model Layer](./protocol-ai-model-layer.md) · [Provider Outbound Gateway](./protocol-provider-gateway.md)

---

Receta paso a paso para agregar una API externa al control plane. Aplica a
Meta, MercadoLibre, Shopify, Tiendanube, SendGrid, MercadoPago, Stripe, OCR,
o cualquier integracion externa.

**Tu trabajo**: declarar el manifest, implementar el OAuth especifico, escribir
la logica de negocio. **Todo lo demas sale gratis.**

---

## La regla mental antes de arrancar

```txt
ENTRADA  (webhooks que el provider envia)
   provider → /api/webhooks/:slug → webhookGateway → RawProviderEvent (PENDING)
                                                            │
                                                            ▼
                                                      worker async procesa

SALIDA   (requests que enviamos al provider)
   modulo → providerHttpGateway.call() → API externa
              ├─ getValidToken (refresh auto)
              ├─ idempotency check
              ├─ rate limit check
              ├─ breaker check
              ├─ retry + backoff
              └─ persistir ProviderHttpAttempt → ApiCostLedger (auto)

DECISION (acciones con riesgo, costo o impacto operativo)
   modulo → AI Routing Decision Engine → RouterDecision
              ├─ Policy Engine (reglas duras)
              ├─ Decision Cache
              ├─ Provider Selector (modelo mas barato suficiente)
              ├─ Schema Validator
              ├─ Business Validator
              └─ ApprovalRequest si corresponde
```

Reglas globales:

```txt
Nunca: AI -> Provider externo directo
Nunca: modulo de negocio -> axios/fetch directo -> Provider externo
Nunca: accion sensible -> executor sin RouterDecision vigente

Siempre:
  webhookGateway para entrada
  AI Routing Decision Engine para decidir
  ApprovalRequest si requiere humano
  providerHttpGateway para salida
  worker/EventOutbox para async + retry
  AuditLog + ApiCostLedger + ProviderHttpAttempt para trazabilidad
```

---

## La receta — 6 pasos

### Paso 1: Manifest

Archivo: `modules/providers/registry/manifests/<slug>.manifest.js`

Declara que es la API y como se habla con ella. Es el unico lugar donde viven:
URL base, scopes, firma del webhook, rate limit, costo, retry policy.

```js
'use strict';
const { CAPABILITIES } = require('../providerCapability.registry');

module.exports = {
  provider:    '<slug>',     // 'sendgrid' | 'tiendanube' | 'ebay' | ...
  apiName:     'api_v1',
  description: 'Descripcion humana.',

  capabilities: [
    CAPABILITIES.SEND_EMAIL,   // agregar a providerCapability.registry si no existe
  ],

  requiredScopes: ['mail.send'],
  optionalScopes: [],

  webhookEvents:         ['delivered', 'opened', 'bounced'],
  signatureScheme:       'hmac',          // 'hmac' | 'jwt' | 'ip_allowlist' | 'none'
  signatureSecretEnvVar: 'PROVIDER_WEBHOOK_SECRET',
  verifyTokenEnvVar:     null,

  rateLimitPolicy: { perSecond: 10, perDay: 100000 },

  costModel: {
    hasVariableCost: true,
    perCallCostUSD:  0.001,
    perUnit:         'email_sent',
  },

  http: {
    baseUrl:    'https://api.example.com',
    authHeader: 'Authorization',
    authFormat: 'Bearer {accessToken}',
    timeoutMs:  20000,
    retryPolicy: {
      maxAttempts: 3,
      backoffMs:   [500, 2000, 5000],
      retryOn:     [408, 429, 500, 502, 503, 504],
    },
  },

  tokenRefresh: {
    refresh: require('../../../<provider>/auth/<provider>OAuth.refresh'),
  },

  supportsSandbox:        true,
  supportsBackfill:       false,
  supportsReconciliation: false,
};
```

Ejemplo de provider marketplace (eBay) con sandbox y base URL condicional:

```js
module.exports = {
  provider: 'ebay',
  apiName:  'sell_commerce_v1',
  capabilities: [CAPABILITIES.READ_ORDERS, CAPABILITIES.SEND_MESSAGE, CAPABILITIES.PUBLISH_PRODUCT],
  requiredScopes: [
    'https://api.ebay.com/oauth/api_scope/sell.inventory',
    'https://api.ebay.com/oauth/api_scope/sell.fulfillment',
    'https://api.ebay.com/oauth/api_scope/commerce.message',
  ],
  webhookEvents:         ['ORDER_CONFIRMATION', 'NEW_MESSAGE', 'MARKETPLACE_ACCOUNT_DELETION'],
  signatureScheme:       'ebay_notification',
  signatureSecretEnvVar: 'EBAY_NOTIFICATION_SECRET',
  rateLimitPolicy:       { perMinute: 100 },
  costModel:             { hasVariableCost: false },
  supportsSandbox:       true,
  supportsBackfill:      true,
  http: {
    baseUrl: process.env.EBAY_ENV === 'sandbox'
      ? 'https://api.sandbox.ebay.com'
      : 'https://api.ebay.com',
    // ...
  },
  tokenRefresh: {
    refresh: require('../../../ebay/auth/ebayOAuth.refresh'),
  },
};
```

Agregar al array de `manifests/index.js` para que se autorregistre.

---

### Paso 2: Refresher OAuth (si la API usa OAuth)

Archivo: `modules/<provider>/auth/<provider>OAuth.refresh.js`

Funcion async de ~20 lineas. `providerAccount.getValidToken()` la invoca
automaticamente cuando detecta que el token esta por vencer.

```js
'use strict';
const axios = require('axios');

module.exports = async function refresh({ refreshToken }) {
  const { data } = await axios.post(
    'https://api.example.com/oauth/token',
    {
      grant_type:    'refresh_token',
      refresh_token: refreshToken,
      client_id:     process.env.EXAMPLE_CLIENT_ID,
      client_secret: process.env.EXAMPLE_CLIENT_SECRET,
    },
    { timeout: 15000 },
  );
  return {
    accessToken:  data.access_token,
    refreshToken: data.refresh_token,   // muchos providers rotan el refresh
    expiresInSec: data.expires_in,
  };
};
```

**Concurrencia: ya esta resuelta en el wrapper `getValidToken`**.

`providerAccount.getValidToken` aplica un lease distribuido (campos
`refreshLockedBy` + `refreshLockExpiresAt` en `ProviderAccount`, TTL ~30s)
ANTES de invocar al refresher. Tu refresher se llama single-shot, no
necesita preocuparse por la carrera.

Si dos workers detectan token vencido al mismo tiempo, solo uno entra al
refresher; el otro pollea y re-lee tokens frescos desde BD. Ver el
`protocol-provider-gateway.md` §"Lease anti-concurrente para refresh"
para el patron completo.

**Rotacion de refreshToken**: si el provider rota (MercadoLibre, OAuth
standard), devolve siempre el `refresh_token` del response. Si no llega,
lanza error ruidoso — la cuenta queda en estado peligroso si el provider
espera rotacion y no la haces.

**Sin rotacion** (Meta long-lived tokens, fb_exchange_token): devolve el
nuevo accessToken tambien como `refreshToken` para que el siguiente
refresh lo use:

```js
return {
  accessToken:  data.access_token,
  refreshToken: data.access_token,   // sin rotacion: re-uso del access
  expiresInSec: data.expires_in,
};
```

Si la API usa API key estatica, omitir este paso — `getValidToken` devuelve
el token guardado directamente.

---

### Paso 3: OAuth controller

Archivos:

```txt
modules/<provider>/controllers/<provider>Integration.controller.js
modules/<provider>/routes/<provider>Integration.routes.js
```

Tres endpoints minimos:

```txt
GET  /api/integrations/<provider>/connect      — genera URL de autorizacion
GET  /api/integrations/<provider>/callback     — recibe code, intercambia por tokens
GET  /api/integrations/<provider>/status       — estado de la cuenta conectada
POST /api/integrations/<provider>/disconnect   — revoca y elimina ProviderAccount
```

El callback siempre guarda con `providerAccount.upsert()` — cifra automaticamente:

```js
await providerAccount.upsert({
  clientId:          clientId,
  provider:          '<slug>',
  providerAccountId: data.user_id,
  displayName:       data.account_name || '',
  scopes:            (data.scope || '').split(' '),
  permissions:       (data.scope || '').split(' '),
  accessToken:       data.access_token,
  refreshToken:      data.refresh_token,
  expiresAt:         new Date(Date.now() + data.expires_in * 1000),
  metadata:          { /* datos especificos del provider */ },
  req,
});
```

Para API key (no OAuth), el connect recibe `apiKey` via POST:

```js
await providerAccount.upsert({
  clientId, provider: '<slug>',
  providerAccountId: fromEmail,
  accessToken: apiKey,  // cifrado automaticamente
  req,
});
```

---

### Paso 4: Webhook endpoint

El controller hace una sola cosa: llamar `webhookGateway.ingest()` y responder
200 inmediato. Nunca hacer trabajo pesado aca.

```js
'use strict';
const webhookGateway = require('../../providers/services/webhookGateway.service');

async function receive(req, res) {
  res.sendStatus(200);  // responder inmediato — antes de procesar

  await webhookGateway.ingest({
    provider:        '<slug>',
    rawBody:         req.rawBody,   // Buffer para validar firma HMAC
    headers:         req.headers,
    payload:         req.body,
    externalEventId: req.body?.notificationId || req.body?.eventId,
    topic:           req.body?.topic,
    onIngested:      async (rawEvent) => {
      await worker.enqueue(rawEvent._id);
    },
  });
}

module.exports = { receive };
```

`webhookGateway.ingest()` verifica la firma segun `manifest.signatureScheme`,
deduplica por `(provider, externalEventId)`, persiste `RawProviderEvent`
y llama `onIngested` si el evento es nuevo.

Para challenge GET (Meta, eBay):

```js
async function receive(req, res) {
  if (req.method === 'GET' && req.query.challenge_code) {
    return res.json({ challengeResponse: computeChallenge(req.query.challenge_code) });
  }
  res.sendStatus(200);
  await webhookGateway.ingest({ ... });
}
```

Para providers sin HMAC (ej: MercadoLibre): el webhook solo persiste y
responde 200. El worker luego re-fetcha el recurso con el access token del
tenant para validar autenticidad.

Registrar en el router de webhooks:

```txt
POST /api/webhooks/<slug>
GET  /api/webhooks/<slug>   (si usa challenge GET)
```

---

### Paso 5: Service + Worker async

**Service** — logica de negocio, usa Decision Engine para decidir y gateway para salir:

```js
'use strict';
const httpGateway     = require('../../providers/services/providerHttpGateway.service');
const providerAccount = require('../../providers/services/providerAccount.service');
const routingDecision = require('../../ai/decisioning/services/routingDecision.service');
const approvalService = require('../../approvals/services/approval.service');

async function execute({ clientId, data }) {
  const account = await providerAccount.findOne({ clientId, provider: '<slug>' });
  if (!account) return { ok: false, error: 'account_not_connected' };

  // Toda accion sensible pasa por el Decision Engine
  const decision = await routingDecision.decide({
    clientId,
    action: {
      type:         'inbound_reply',   // ver action.type canonicos
      sourceModule: '<provider>',
      sourceType:   '<action_type>',
      sourceRef:    String(data._id),
    },
    channel: {
      channelType:   '<slug>',
      channelStatus: account.status === 'connected' ? 'connected' : 'disconnected',
    },
    mode: 'advisory',
  });

  if (decision.decision === 'block') {
    return { ok: false, error: 'decision_blocked', decisionId: decision.decisionId };
  }

  if (decision.requiresApproval) {
    const approval = await approvalService.requestApproval({ clientId, sourceModule: '<provider>', ... });
    return { ok: false, pendingApproval: true, approvalId: approval._id };
  }

  // Llamar API externa SOLO con decision vigente
  const result = await httpGateway.call({
    account,
    method: 'POST',
    path:   '/v1/<endpoint>',
    body:   { /* payload */ },
    idempotencyKey: `<slug>_<action>_${data._id}_v1`,
    sourceModule:   '<provider>',
    sourceType:     '<action_type>',
    sourceId:       String(data._id),
  });

  return result;
}

module.exports = { execute };
```

**Worker** — pollea `RawProviderEvent` y lo procesa:

```js
'use strict';
const RawProviderEvent = require('../../providers/models/RawProviderEvent.model');
const service          = require('../services/<provider>.service');
const logger           = require('../../../infrastructure/logger/logger');

const POLL_INTERVAL_MS = 30 * 1000;
const BATCH_SIZE = 20;

async function processBatch() {
  const events = await RawProviderEvent.find({
    provider: '<slug>',
    status:   RawProviderEvent.STATUS.PENDING,
  }).sort({ receivedAt: 1 }).limit(BATCH_SIZE);

  for (const event of events) {
    try {
      const claimed = await RawProviderEvent.findOneAndUpdate(
        { _id: event._id, status: RawProviderEvent.STATUS.PENDING },
        { $set: { status: RawProviderEvent.STATUS.PROCESSING }, $inc: { attempts: 1 } },
        { new: true },
      );
      if (!claimed) continue;

      await service.handleWebhook(event.payload, event);

      await RawProviderEvent.updateOne(
        { _id: event._id },
        { $set: { status: RawProviderEvent.STATUS.PROCESSED, processedAt: new Date() } },
      );
    } catch (err) {
      logger.error(`[<slug>-worker] ${event._id} fallo: ${err.message}`);
      await RawProviderEvent.updateOne(
        { _id: event._id },
        {
          $set: {
            status: event.attempts >= 5
              ? RawProviderEvent.STATUS.DEAD_LETTERED
              : RawProviderEvent.STATUS.PENDING,
            error: err.message.slice(0, 500),
          },
        },
      );
    }
  }
}

function start() {
  setInterval(() => {
    processBatch().catch((err) => logger.error(`[<slug>-worker] poll fallo: ${err.message}`));
  }, POLL_INTERVAL_MS);
}

module.exports = { start, processBatch };
```

El worker sigue este orden interno:

```txt
1. Lee RawProviderEvent (claim atomico)
2. Re-fetch del resource via providerHttpGateway — no confiar en el payload del webhook
3. Normaliza al modelo de dominio
4. Arma RoutingSnapshot via Context Builder
5. Llama routingDecision.service.decide(snapshot)
6. Segun resultado: crea ApprovalRequest, notifica, o ejecuta
```

---

### Paso 6: Context Builder y RoutingSnapshot

Archivo: `modules/<dominio>/context/<dominio>Context.builder.js`

Traduce datos del dominio al contrato universal con `action.type`:

```js
'use strict';
const RoutingSnapshot = require('../../routing/RoutingSnapshot');

async function buildSnapshot({ clientId, provider, actionType, sourceType, sourceId, account }) {
  const factory = actionType === 'content_publish'
    ? RoutingSnapshot.forContentPublish
    : RoutingSnapshot.forInboundReply;

  return factory({
    clientId,
    sourceId,
    action: {
      type:         actionType,
      sourceModule: '<dominio>',
      sourceType,
      sourceRef:    sourceId,
    },
    channel: {
      channelType:   provider,
      channelStatus: account.status,
    },
    mode: 'enforced',
  });
}

module.exports = { buildSnapshot };
```

Mapeo de casos a `action.type`:

```txt
Respuesta a mensaje/pregunta entrante  →  action.type = 'inbound_reply'
Publicacion en canal externo           →  action.type = 'content_publish'
Envio masivo de campaña                →  action.type = 'campaign_send'
Respuesta automatica del bot           →  action.type = 'auto_reply'
Moderacion de comentario               →  action.type = 'comment_moderate'
```

`action.type` determina el scope de policy. `sourceType` es solo metadata de
auditoria.

---

### Paso 6b: Registrar features en providerSelector (solo si usa IA)

```js
// providerSelector.service.js — array CANDIDATES
{ provider: 'groq',       model: 'llama-3.1-8b-instant',    tier: 'cheap',    supports: ['<slug>_classify'], riskCeiling: 'low' },
{ provider: 'openrouter', model: 'openai/gpt-4o-mini',       tier: 'balanced', supports: ['<slug>_compose'],  riskCeiling: 'medium' },
```

Criterio:

```txt
Clasificacion simple, riesgo bajo      → cheap (llama-8b)
Generacion de contenido publico        → balanced (gpt-4o-mini)
Decisiones de alto riesgo / reputacion → premium (gpt-4o o claude-opus)
```

---

## Que sale gratis vs que escribis

| Capacidad | Vos escribis | Sale gratis |
|---|---|---|
| Recibir webhooks | Manifest con `signatureScheme` | `/api/webhooks/:slug` valida firma, deduplica, persiste, responde 200 |
| Cuenta conectada | OAuth controller | `ProviderAccount.upsert()` cifra automatico |
| Token fresco | Refresher OAuth (~20 lineas) | `getValidToken()` refresca automatico |
| Llamar API externa | Logica de negocio | `providerHttpGateway.call()`: idempotency, retry, breaker, headers, timeouts |
| Idempotencia | Pasar `idempotencyKey` | `ProviderHttpAttempt` unique index |
| Retry + backoff | Manifest con `retryPolicy` | Gateway aplica backoff exponencial |
| Circuit breaker | Nada | `providerBreaker` corta endpoint si falla repetidamente |
| Costo trackeado | Manifest con `costModel` | `ApiCostLedger` se escribe automatico |
| Decidir con IA | Feature en `providerSelector.CANDIDATES` + RoutingSnapshot | Decision Engine: policy, cache, selector, schema, business validator, fallback |
| Aprobacion humana | `approvalService.requestApproval()` | Modelo unificado, UI generica, dedupe |
| Webhook async | Worker que pollea `RawProviderEvent` | Outbox + reintentos + DLQ |

---

## Action types canonicos

Antes de crear uno nuevo, revisar si ya existe uno equivalente:

```txt
campaign_send               inbound_reply             content_publish
comment_moderate            auto_reply
marketplace_answer_question marketplace_publish_product marketplace_sync_stock
mail_transactional_send     payment_capture            ocr_process_document
```

Regla:

```txt
action.type nombra la intencion de negocio, no el endpoint externo.

Bien: marketplace_answer_question
Mal:  post_answers
```

---

## Idempotency keys canonicas

```txt
<provider>_<accion>_<id>_v<version>

Instagram DM reply:       ig_dm_reply_<eventId>_v1
Instagram comment hide:   ig_comment_hide_<eventId>_v1
MELI answer:              ml_question_answer_<questionId>_v1
MELI stock sync:          ml_stock_sync_<itemId>_<newStock>_v1
MELI publish item:        ml_publish_item_<productId>_v1
eBay message reply:       ebay_message_reply_<conversationId>_v1
Mail transaccional:       mail_transactional_<kind>_<sourceId>
Payment capture:          payment_capture_<invoiceId>_v1
```

La key debe ser estable para la misma accion logica. Debe cambiar si el
contenido cambia (usar version o hash del payload).

---

## Definition of Done para una API nueva

Un canal no esta completo hasta que cumple:

- [ ] Manifest en `providers/registry/manifests/` registrado en `index.js`
- [ ] Usa `ProviderAccount` para cuentas/tokens/status
- [ ] Si usa OAuth, tiene refresher (la concurrencia la maneja el lease de `getValidToken`)
- [ ] Webhooks entran por `webhookGateway` + persisten en `RawProviderEvent`
- [ ] Worker procesa `RawProviderEvent` de forma idempotente
- [ ] Worker hace re-fetch del resource, no confía solo en el payload
- [ ] Toda llamada saliente usa `providerHttpGateway.call()`
- [ ] Acciones sensibles tienen `RouterDecision` vigente
- [ ] Acciones que requieren humano usan `ApprovalRequest`
- [ ] Idempotency keys canonicas por tipo de accion
- [ ] Costos variables en `ApiCostLedger`
- [ ] Llamadas externas en `ProviderHttpAttempt`
- [ ] `AuditLog` registra connect/disconnect/acciones sensibles
- [ ] Tests cubren token vencido, retry, idempotencia, policy block y webhook duplicado
- [ ] UI muestra estados: conectado, faltan permisos, token vencido, rate limited

---

## Checklist antes de ir a produccion

```txt
[ ] Manifest registrado en index.js
[ ] ProviderAccount guarda tokens cifrados
[ ] OAuth callback funciona en sandbox con cuenta real
[ ] Refresh de tokens single-shot bajo concurrencia (lease del wrapper)
[ ] Webhook responde 200 en < 5s (< 500ms para MELI)
[ ] Webhook deduplica por externalEventId
[ ] Worker re-fetch usa providerHttpGateway, no axios directo
[ ] Idempotency key estable por accion logica
[ ] RouterDecision se persiste por cada decision
[ ] ApprovalRequest se crea cuando corresponde
[ ] Executor no ejecuta sin RouterDecision vigente
[ ] ProviderHttpAttempt se persiste en cada llamada
[ ] AuditLog no contiene tokens ni secretos
[ ] Circuit breaker abierto no llama al provider
[ ] Rate limit excedido no llama al provider
[ ] 429/5xx reintenta con backoff
[ ] 4xx no retryable no reintenta
[ ] Feature flag en false = modulo invisible
[ ] Tests cubren OAuth, webhook, idempotencia, approval y envio
```

---

## Cuando NO seguir esta receta

- **APIs de embeddings**: no son decisiones de negocio. Usan AI Usage Ledger
  pero no necesitan `RouterDecision` salvo que disparen una accion posterior.
- **APIs internas** (entre microservicios propios): no son providers externos.
  No usan `providerHttpGateway`.
- **Canales con sesion persistente** (WebSocket/socket.io, Baileys): el ciclo
  de vida es distinto. No forzar el patron de provider sobre conexiones vivas.
- **APIs de solo lectura sin OAuth ni costo** (ej: tipo de cambio): pueden
  empezar como helper interno. Si tienen rate limit, token, costo o afectan
  un workflow, crear manifest y usar `providerHttpGateway`.

Regla de desempate:

```txt
Si la llamada externa puede fallar y afectar una accion visible del usuario,
debe pasar por providerHttpGateway.
```

---

## Panel ROOT — troubleshooting durante implementacion

Mientras implementas una API nueva, el panel ROOT en `/admin/providers`
es la herramienta principal de debugging. Sin necesidad de leer logs ni
tocar la BD, podes ver:

```txt
Cuentas             ProviderAccount con status, expiresAt, lastError.
                    Detectas tokens vencidos antes de que el flujo falle.

Llamadas HTTP       ProviderHttpAttempt con filtros por provider/clientId/status.
                    Modal con responseSnapshot, attempts, latencyMs, errorCode.

Webhooks recibidos  RawProviderEvent con boton "Reintentar" en eventos
                    dead_lettered/failed. Recovery operativa sin acceso a BD.

Providers reg.      Cards con manifests (capabilities, webhooks, costo).
```

Endpoint del retry: `POST /admin/providers/raw-events/:id/retry`.
Acepta solo estados retryables (`dead_lettered`, `failed`); el resto
devuelve 409 `invalid_status`.

Ver el protocolo de Provider Outbound Gateway §"Admin / ROOT" para el
listado completo de endpoints sugeridos.

## Helpers de tests reusables

Despues de escribir 3+ archivos de test del framework, los mocks se repiten.
Centralizar fixtures en `tests/_fixtures/providers.js` (prefijo `_` para
excluir del testMatch del runner) reduce ~30% del codigo de cada test:

```txt
IDS                          ObjectIds canonicos
makeProviderAccount({...})   stub completo con .save() y campos de lease
makeManifest({...})          manifest con retry corto (10/20/30ms) para tests rapidos
callParams({...})            params canonicos para providerHttpGateway.call()
mockGetValidToken({...})     shape estandar { accessToken, account, refreshed }
mockAxiosOk(data, status)    response 2xx
mockAxiosError(status, data) response 4xx/5xx
mockAxiosTimeout()           Error con code ECONNABORTED
mockResLike()                Express res stub encadenable
```

Patron: la primera vez que escribis un mock, no lo extraigas. La segunda
vez que se repite, moveio al fixture. La tercera vez ya deberia estar ahi.

## Lo que NO hay que hacer

```txt
No crear un cliente HTTP propio por provider (no EbayClient, no MLClient).
No llamar axios directo desde un service de dominio.
No guardar tokens en texto plano ni en logs.
No hacer trabajo pesado en el controller del webhook.
No confiar en el payload del webhook como fuente completa — siempre re-fetch.
No activar enforced sin schema validator, business validator e idempotencia.
No usar canal experimental como fallback de uno oficial.
```

---

## Si el caso no encaja en la receta

Si agrega un caso que no encaja en este patron (nuevo tipo de webhook,
OAuth con flujo distinto, middleware adicional), documentar la excepcion
en este mismo archivo. Es el indice de referencia del equipo — si queda
desactualizado pierde el valor.
