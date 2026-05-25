# Protocolo: Provider Outbound Gateway

> Parte del [Safe Automation Control Plane](./README.md).
> Relacionado con: [AI Routing Decision Engine](./protocol-decision-engine.md) · [AI Model Layer](./protocol-ai-model-layer.md) · [API Integration](./protocol-api-integration.md)

---

La idea central:

```txt
El modulo de negocio declara la intencion.
El gateway aplica politicas, seguridad y observabilidad.
El provider recibe una llamada controlada, idempotente y auditable.
```

---

## Problema

Sin un gateway comun, cada modulo reimplementa lo mismo:

```txt
modulo de instagram  -> axios.post('https://graph.facebook.com/...')
modulo de marketplace -> axios.get('https://api.mercadolibre.com/...')
modulo de email      -> fetch('https://api.sendgrid.com/...')
```

Ese enfoque produce duplicacion, bugs distintos por provider y poca visibilidad
operativa cuando la llamada puede: consumir saldo, duplicarse por timeout, fallar
por token vencido, disparar cascadas contra un provider caido, o exceder rate limits.

---

## Principio

No llamar una API externa directo desde un modulo de dominio.

Nunca:

```txt
Service de negocio -> axios/fetch -> API externa
```

Siempre:

```txt
Service de negocio
  -> providerHttpGateway.call()
  -> Provider Registry + manifest
  -> ProviderAccount / token refresh
  -> idempotency check
  -> circuit breaker
  -> rate limit
  -> retry/backoff
  -> ProviderHttpAttempt (audit)
  -> ApiCostLedger (si aplica)
  -> API externa
```

---

## Uso

```js
const result = await providerHttpGateway.call({
  account,
  method: 'POST',
  path:   `/${igUserId}/messages`,
  body: {
    recipient: { id: recipientId },
    message:   { text: 'Hola' },
  },
  idempotencyKey: `ig_dm_${eventId}`,
  sourceModule:   'instagram',
  sourceType:     'dm_send',
  sourceId:       String(eventId),
});

if (!result.ok) {
  // token_unavailable, circuit_open, rate_limited, http_4xx, timeout, etc.
}
```

Respuesta uniforme:

```txt
ok, status, data, error
attempts, latencyMs
refreshedToken
correlationId, attemptId, apiCostLedgerId
```

El gateway nunca lanza excepciones por fallos del provider. Devuelve
`{ ok: false, error }` y deja que el caller decida.

---

## Componentes

### Provider Registry

Cada provider declara un manifest con capacidades, scopes, events de webhook,
firma de webhook, rate limit, costo y configuracion HTTP.

```txt
Agregar provider nuevo = agregar manifest nuevo.
No cambiar el gateway.
```

### Manifest HTTP

```js
module.exports = {
  provider: 'mercadolibre',
  apiName:  'api_v1',

  requiredScopes: ['offline_access', 'read', 'write'],
  webhookEvents:  ['questions', 'orders_v2', 'items'],
  signatureScheme: 'ip_allowlist',  // MELI no usa HMAC
  rateLimitPolicy: { perHour: 3600 },
  costModel: { hasVariableCost: false },

  http: {
    baseUrl:    'https://api.mercadolibre.com',
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
    refresh: require('../../../marketplace/auth/mercadolibreOAuth.refresh'),
  },
};
```

El caller no conoce `baseUrl`, formato de auth, timeout ni retry policy.

### ProviderAccount

Modelo unico para cuentas conectadas. Guarda tokens cifrados, scopes, permisos,
estado, `expiresAt` y metadata provider-specific.

Estados:

```txt
not_connected | connecting | connected | missing_permissions
token_expired | restricted | disabled | disconnected | error
```

Regla: los tokens nunca se devuelven a UI, logs o respuestas API.

### getValidToken

Devuelve un access token usable:

```txt
1. Token vivo → devolverlo
2. Token vencido o por vencer → llamar manifest.tokenRefresh.refresh
3. Persistir tokens nuevos cifrados
4. Si falla → marcar cuenta token_expired
5. Si no hay token → devolver token_unavailable
```

### Idempotencia

Cuando el caller pasa `idempotencyKey`, el gateway revisa si ya existe un
`ProviderHttpAttempt` exitoso. Si existe, no llama al provider y devuelve
el resultado persistido.

```txt
Index unique: (provider, idempotencyKey)
```

Formato recomendado:

```txt
<provider>_<accion>_<id>_v<version>

ig_dm_reply_<eventId>_v1
ml_question_answer_<questionId>_v1
ml_stock_sync_<itemId>_<newStock>_v1
shopify_inventory_update_<variantId>_v1
```

### ProviderHttpAttempt

Registro auditable de cada llamada saliente.

```txt
clientId, provider, apiName, providerAccountId
method, endpoint (normalizado con placeholders, no IDs concretos)
idempotencyKey, sourceModule, sourceType, sourceId
status: pending | success | error | timeout | circuit_open | rate_limited | token_unavailable
httpStatus, responseSnapshot (sanitizado, sin tokens)
errorCode, errorMessage
attempts, refreshedToken
costEstimateUSD, apiCostLedgerId
latencyMs, correlationId, occurredAt
```

Indices:

```txt
unique(provider, idempotencyKey) cuando existe
clientId + provider + occurredAt
sourceModule + sourceType + sourceId + occurredAt
provider + status + occurredAt
occurredAt TTL
```

### Circuit Breaker

Scope: por `(provider, endpoint)`. Meta `/messages` puede fallar mientras
`/media` funciona; cortarlos juntos seria incorrecto.

```txt
closed    → llamadas normales
open      → devuelve circuit_open sin llamar al provider
half-open → un request de prueba; si pasa vuelve a closed, si falla vuelve a open
```

### Rate Limit

Por `(clientId, provider)`, consumiendo `manifest.rateLimitPolicy`.

Si excede: `rate_limited`, no llama al provider.

Implementacion inicial: token bucket in-memory.
Al escalar: Redis token bucket atomico.

### Retry y backoff

Reglas por `manifest.retryPolicy`:

```txt
2xx           → success
4xx permanente → no reintentar
408/429/5xx   → backoff y reintentar hasta maxAttempts
timeout       → reintentar hasta maxAttempts
```

### ApiCostLedger

Se escribe cuando `manifest.costModel.hasVariableCost = true` o
`manifest.costModel.perCallCostUSD > 0`.

---

## Refresh de tokens — contrato del refresher

```js
module.exports = async function refresh({
  provider, providerAccountId, refreshToken, scopes, metadata
}) {
  return {
    accessToken:  'new-access-token',
    refreshToken: 'new-refresh-token',  // si el provider rota
    expiresInSec: 3600,
  };
};
```

Reglas:

```txt
No loguear tokens
No persistir tokens sin cifrar
Si falla, marcar cuenta token_expired
Si el provider rota refreshToken (MELI), usar lease en metadata para evitar refresh concurrente
```

Lease para tokens single-use (ej: MercadoLibre):

```txt
metadata.refreshLockedUntil = now + 30s antes del call HTTP
finally: unset refreshLockedUntil
Si refreshLockedUntil > now → throw 'refresh locked, otro worker esta refrescando'
```

---

## Webhook entrante — webhookGateway

El gateway inbound es el espejo de entrada del outbound gateway.

```txt
POST /api/webhooks/:provider
  -> validar firma HMAC (si el provider la soporta)
  -> deduplicar por (provider, externalEventId)
  -> persistir RawProviderEvent (status: pending)
  -> responder 200 en < 5s (< 500ms para MELI)
  -> worker async procesa RawProviderEvent
```

Para providers sin HMAC (ej: MELI):

```txt
validacion = IP allowlist + re-fetch del recurso desde el worker
el webhook solo persiste y responde 200
el worker hace GET {resource} con el token del tenant para validar autenticidad
```

---

## Como agregar un provider nuevo

```txt
1. Crear manifest en providers/registry/manifests/<slug>.manifest.js
2. Registrar en manifests/index.js
3. Crear refresher OAuth si el provider usa refresh tokens
4. Crear controller OAuth provider-specific (connect/callback/disconnect)
5. Guardar cuenta con ProviderAccount.upsert()
6. Llamar APIs externas con providerHttpGateway.call()
7. Configurar webhookGateway si el provider envia eventos
```

No crear de nuevo:

```txt
HTTP client con retry
token refresh generico
idempotencia
circuit breaker
rate limit
cost ledger
ProviderHttpAttempt
```

---

## Estructura de archivos sugerida

```txt
src/
  modules/
    providers/
      registry/
        providerCapability.registry.js
        manifests/
          index.js
          meta_whatsapp.manifest.js
          meta_instagram.manifest.js
          mercadolibre.manifest.js
          shopify.manifest.js
      models/
        ProviderAccount.model.js
        ProviderHttpAttempt.model.js
        RawProviderEvent.model.js
      services/
        providerAccount.service.js
        providerHttpGateway.service.js
        providerBreaker.service.js
        providerRateLimit.service.js
        webhookGateway.service.js
      routes/
        providersWebhook.routes.js
    costs/
      models/
        ApiCostLedger.model.js
```

---

## Errores uniformes

```txt
token_unavailable     | circuit_open         | rate_limited
timeout               | network_error
http_400 | http_401 | http_403 | http_404 | http_429 | http_500
```

El caller no parsea errores nativos de Axios, Fetch ni SDKs externos.

---

## Checklist de implementacion

Primer corte:

- [ ] `ProviderAccount.model` + `ProviderHttpAttempt.model`
- [ ] Provider Registry + manifests iniciales
- [ ] `providerAccount.getValidToken` con refresh automatico
- [ ] `providerBreaker` por `(provider, endpoint)`
- [ ] `providerRateLimit` por `(clientId, provider)`
- [ ] `providerHttpGateway.call` con idempotencia, retry, audit
- [ ] `ApiCostLedger` integrado
- [ ] Tests: token vigente, token vencido, idempotencia, circuit open, rate limited

Segundo corte:

- [ ] Refreshers OAuth provider-specific
- [ ] OAuth controllers (connect/callback/disconnect)
- [ ] Healthcheck de cuentas conectadas
- [ ] `webhookGateway` con deduplicacion por `(provider, externalEventId)`
- [ ] `RawProviderEvent.model` + workers de ingest
- [ ] Metricas
- [ ] Vista admin de attempts

Tercer corte:

- [ ] Rate limit Redis para multiples workers
- [ ] Lint/regla de review contra llamadas directas a providers
- [ ] Reconciliacion de costos reales
- [ ] Backfill/sync con cursor provider-specific

---

## Tests — casos minimos

```txt
Token vigente:            no llama refresher, llama provider
Token vencido + refresher: llama refresher, persiste token nuevo, llama provider
Token vencido sin refresher: devuelve token_unavailable, no llama provider
Idempotency hit:          no llama provider, devuelve resultado persistido
Circuit breaker open:     devuelve circuit_open, no llama provider
Rate limit excedido:      devuelve rate_limited, no llama provider
429 retryable:            reintenta hasta success o maxAttempts
500 retryable:            reintenta con backoff
400 no retryable:         no reintenta
Timeout:                  reintenta, si agota devuelve timeout
Response grande:          persiste snapshot truncado, sin tokens
Costo variable:           crea ApiCostLedger vinculado al attempt
```

---

## Bugs comunes

### Idempotency key demasiado generica

```txt
Malo:  sourceId
Mejor: provider + action + sourceId + version
```

### Endpoint con IDs concretos en el attempt

```txt
Malo:  /123456/messages          (no agrupa metricas)
Mejor: /{igUserId}/messages      (normalizado)
```

### Loguear headers

Los headers contienen tokens. Nunca persistir `Authorization`, `accessToken`,
`refreshToken` ni secretos en logs ni en `responseSnapshot`.

### Reintentar 4xx permanentes

Un 400 por payload invalido no se arregla con retry. Solo `retryOn` explicito
desde el manifest.

### Circuit breaker global por provider

Si un endpoint falla y se corta todo el provider, se pierde capacidad
innecesariamente. Scope: `(provider, endpoint)`.

---

## Observabilidad

```txt
provider_http_requests_total{provider, endpoint, status}
provider_http_latency_ms{provider, endpoint}
provider_http_retries_total{provider, endpoint}
provider_http_circuit_open_total{provider, endpoint}
provider_http_rate_limited_total{provider}
provider_http_token_unavailable_total{provider}
provider_http_idempotency_hits_total{provider}
provider_http_cost_estimated_total{provider}
```

Alertas:

```txt
token_unavailable alto por provider
circuit_open sostenido
429 alto en ventana corta
latency p95 alta
rate_limited alto por tenant
```

---

## Relacion con el Decision Engine

```txt
Decision Engine → produce RouterDecision (routable)
Executor → consume RouterDecision
Executor → llama providerHttpGateway.call()
providerHttpGateway → llama API externa
```

La IA no tiene tokens ni URLs de providers. No llama al gateway directamente.

---

## Regla final

```txt
Los modulos de negocio declaran intencion.
El gateway controla la salida.
Cada llamada queda medida, limitada, idempotente y auditable.
```
