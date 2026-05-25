# Protocolo: AI Routing Decision Engine

> Parte del [Safe Automation Control Plane](./README.md).
> Relacionado con: [AI Model Layer](./protocol-ai-model-layer.md) · [Provider Outbound Gateway](./protocol-provider-gateway.md) · [API Integration](./protocol-api-integration.md)

---

La idea central:

```txt
La IA propone.
La plataforma valida.
El executor solo ejecuta decisiones validadas.
```

---

## Problema

Muchas plataformas conectan IA directo a acciones:

```txt
usuario pide algo
  -> modelo decide
  -> sistema ejecuta
```

Ese flujo es peligroso cuando la accion puede:

- generar costo variable;
- enviar mensajes por canales pagos;
- usar canales sensibles o no oficiales;
- publicar contenido;
- afectar reputacion del negocio;
- violar limites de plan;
- duplicar envios;
- ignorar opt-out o consentimiento;
- requerir aprobacion humana.

La solucion es separar decision, validacion y ejecucion.

---

## Principio

No usar IA si una regla, cache o herramienta deterministica puede resolver bien.

Orden recomendado:

```txt
Rule Engine / Policy Engine
  -> Decision Cache
  -> AI Router si aporta valor
  -> Schema Validator
  -> Business Validator
  -> Approval Workflow
  -> Executor
  -> Audit / Billing / Outbox
```

Nunca:

```txt
AI Router -> Executor directo
```

---

## Quién puede originar una decision

```txt
Usuario via UI del panel       -> crea campaña, publica producto, registra gasto
Bot o automatizacion           -> responde mensaje entrante, auto-reply programado
Asistente IA interno           -> propone via tool propose_*, motor valida
Worker o job programado        -> sync de stock, reconciliacion de ordenes
Webhook de proveedor externo   -> pregunta de marketplace, orden, mensaje inbound
```

Todos los origenes producen el mismo resultado: un `RoutingSnapshot` con
`action.type` declarado. El motor no sabe ni le importa quien armo el snapshot.

---

## action.type — contrato universal

El `action.type` determina el scope de policy, el feature del selector y las
validaciones que aplican. `sourceModule` y `sourceType` son solo metadata de
trazabilidad — no cambian el flujo.

```txt
action.type:
  campaign_send      — envio masivo de campaña
  inbound_reply      — respuesta a mensaje o evento entrante
                       (DMs, preguntas de marketplace, propuestas del asistente)
  content_publish    — publicar producto o post en canal externo
  auto_reply         — respuesta automatica del bot
  comment_moderate   — moderar o responder comentario publico

action.sourceModule:  campaigns | marketplace | social | finance | assistant | crm
action.sourceType:    metadata (marketplace_answer_question, finance_expense_create, etc.)
```

---

## Flujo principal

```txt
RoutingSnapshot (action.type declarado)
  -> Policy Engine               reglas duras — puede bloquear antes de IA
  -> Decision Cache              ¿ya se calculo este contexto?
  -> Provider Selector           elige modelo segun feature, riesgo y plan
  -> AI Router                   llama al LLM
  -> Schema Validator            rechaza output malformado
  -> Business Validator          re-valida reglas con output de IA
  -> ApprovalRequest             si requiresApproval = true
  -> RouterDecision (routable)
  -> Executor
  -> Provider Outbound Gateway
```

---

## Componentes

### Policy Engine

Evalua reglas duras antes de llamar al modelo:

```txt
plan no permite la accion -> block
saldo insuficiente -> block
usuario sin permiso -> block
canal desconectado -> block
destino sin consentimiento -> block
opt-out activo -> block
feature flag apagada -> block
approval requerida y no aprobada -> block
```

La IA no puede ignorar estas reglas.

Scopes por `action.type`:

```txt
router_ai.campaign_send
router_ai.inbound_reply
router_ai.content_publish
router_ai.auto_reply
router_ai.comment_moderate
```

### AI Router

Optimiza dentro de lo permitido. Puede sugerir tandas, delays, riesgo,
alternativas mas baratas. No puede ignorar limites de plan, enviar sin saldo,
usar canales no habilitados ni ejecutar sin aprobacion.

Ver [AI Model Layer](./protocol-ai-model-layer.md) para la seleccion de modelos.

### Schema Validator

Valida output del modelo contra JSON Schema estricto.

```txt
si falla: reintentar una vez
si falla dos veces: ruleOnlyFallback
registrar evento schema_failed
```

`additionalProperties: false` es obligatorio en todos los schemas.

### Business Validator

Vuelve a validar despues de la IA:

```txt
plan, limites, saldo, reserva de costo
permisos, feature flags
estado del canal y proveedor
consentimiento, opt-out, horarios
aprobaciones, circuit breakers
idempotencia, expiracion de la decision
```

Aunque la IA devuelva `allow`, el Business Validator puede convertirlo en `block`.

### Executor

Solo ejecuta decisiones validadas. No recalcula. Si la decision expiro o cambio
un dato critico, bloquea y pide nueva decision.

---

## RouterDecision — modelo de persistencia

```txt
clientId, sourceModule, sourceType, sourceId
decisionId, mode, state, decision
channel, riskLevel, confidence
inputHash, contextHash
schemaVersion, policyVersion, promptVersion
provider, model, tokensInput, tokensOutput
estimatedAiCostUSD, estimatedCostARS
requiresApproval, approvalRequestId, expiresAt
summary, reasonCodes, rulesApplied
batches, perDestinationDecisions, blockedDestinations
requiredActions, balanceReservation
cacheHit, fallbackUsed
rawOutput, validatedOutput, businessValidationResult
correlationId, causationId, idempotencyKey
createdAt, updatedAt
```

Estados:

```txt
requested -> rule_checked -> ai_requested -> ai_decided -> schema_validated
  -> business_validated -> approval_pending -> approved -> routable
  -> blocked | expired | failed | cancelled
```

---

## ruleOnlyFallback

Cuando la IA falla o no aporta valor:

```txt
reglas bloquean:      decision = block
riesgo bajo + reglas pasan: decision = allow (tandas minimas, delays conservadores)
riesgo medio:         decision = require_approval
riesgo alto/critico:  decision = block
```

---

## Patron del asistente interno

```txt
Asistente recibe pedido en lenguaje natural
  -> tools read_* consultan datos (sin pasar por el motor)
  -> tool propose_* arma RoutingSnapshot con action.type correcto
  -> llama routingDecision.service
  -> motor ejecuta flujo completo
  -> propose_* devuelve { status: 'pending_approval' | 'allowed' | 'blocked' }
  -> Asistente comunica resultado al usuario
```

El asistente no ejecuta. No llama APIs externas. No aprueba sus propias propuestas.

Mapeo de acciones a action.type:

```txt
responder pregunta de marketplace  -> action.type = 'inbound_reply', sourceModule = 'marketplace'
publicar producto                  -> action.type = 'content_publish', sourceModule = 'marketplace'
enviar campaña                     -> action.type = 'campaign_send',   sourceModule = 'campaigns'
moderar comentario                 -> action.type = 'comment_moderate', sourceModule = 'social'
registrar gasto                    -> action.type = 'inbound_reply',   sourceModule = 'finance'
```

---

## Canales experimentales

```txt
experimentalChannelEnabled debe ser true
tenant debe tener habilitacion admin/root y aceptar terminos especificos
no usar canal experimental como fallback automatico
no permitir modo enforced sin decision explicita de producto
```

---

## Modos de rollout

| Modo | Comportamiento |
|---|---|
| `simulation` | Calcula decision, no ejecuta |
| `shadow` | Llama al modelo, registra, no afecta produccion |
| `advisory` | Sugiere, humano aprueba |
| `enforced` | Ejecuta automatico con decision vigente y validada |

---

## Estructuras necesarias antes de implementar

```txt
RouterDecision.model + routerDecision.schema
EventOutbox (atomico con RouterDecision)
ApprovalRequest
RoutingSnapshot con action.type
Policy Engine con scopes
AI Decision Cache (key: inputHash + contextHash)
Provider Selector con circuit breaker
AI Circuit Breaker
AI Usage Ledger
{canal}CostEstimator (retorna 0 si falla — nunca rompe el flujo)
{canal}BalanceService (para enforced)
ApprovalService
Rate Card + Exchange Rate Service
```

---

## Checklist de implementacion

Primer corte:

- [ ] `RouterDecision.model` + `routerDecision.schema`
- [ ] `ruleOnlyFallback.service` — el motor opera sin IA desde el dia uno
- [ ] Policy Engine con politicas minimas del canal
- [ ] `routingDecision.service` (orquestador)
- [ ] `RoutingSnapshot` con `action.type` + primer `ChannelSnapshot` adapter
- [ ] Adaptador de dominio (`campaignRouting.service` o equivalente)
- [ ] Endpoint de simulacion
- [ ] Tests de los casos minimos obligatorios

Segundo corte:

- [ ] EventOutbox atomico con RouterDecision
- [ ] Decision Cache + Provider Selector
- [ ] AI Usage Ledger + Circuit Breaker
- [ ] Approval Workflow
- [ ] Shadow mode
- [ ] Metricas con label `{channel}`

Tercer corte:

- [ ] Advisory para tenants beta
- [ ] Enforced solo en canales estables
- [ ] Nuevos `ChannelSnapshot` adapters para canales adicionales

---

## Tests — casos minimos obligatorios

```txt
Plan insuficiente            -> block + PLAN_NOT_ALLOWED
Canal desconectado           -> block + CHANNEL_DISCONNECTED
Saldo insuficiente           -> block + CHANNEL_BALANCE_INSUFFICIENT
Opt-out total                -> block + OPT_OUT
Opt-out parcial              -> decision != block, blockedDestinations con opt-out
IA no disponible             -> ruleOnlyFallback, fallbackUsed=true, sin excepcion
Schema invalido 2 veces      -> ruleOnlyFallback, schema_failed emitido
IA allow + Business Validator block -> decision=block
Cache hit                    -> segunda request no llama a la IA, cacheHit=true
Cache invalidada por contexto -> nueva llamada a la IA
Canal experimental enforced  -> block sin importar IA
Decision expirada            -> executor rechaza, no ejecuta
```

---

## Bugs reales encontrados al implementar

### isIso aceptaba strings no-ISO

`Date.parse()` de V8 acepta strings que no son ISO 8601. Fix:

```js
const ISO_RE = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/;
function isIso(v) {
  return isString(v) && ISO_RE.test(v) && !Number.isNaN(Date.parse(v));
}
```

### Policy Engine no bloqueaba en tests

Las politicas retornaban `allowed: true` porque nadie llamaba `registerAll()`.
Agregar en `beforeAll`: `registerAll()`.

### Modelos Groq que no existen

Nombres de marketing no coinciden con IDs reales de la API. Verificar siempre
contra `groq.models.list()` antes de agregar al registry.

### IA genera expiresAt en el pasado

El modelo usa fechas de su training cutoff. Fix: normalizar server-side + informar
fecha actual en el system prompt: `La fecha y hora actual es: ${new Date().toISOString()}`.

### tenantId como ObjectId en eventos del outbox

El event contract espera string. Fix: `tenantId: String(snapshot.clientId)`.

---

## Observabilidad

```txt
ai_router_requests_total{channel}
ai_router_cache_hits_total{channel}
ai_router_fallback_total
ai_router_schema_failures_total
ai_router_cost_usd
ai_router_tokens_input_total
ai_router_tokens_output_total
router_ai_shadow_disagreements_total
router_ai_decision_expired_total
router_ai_balance_reservation_failures_total{channel}
```

---

## Regla final

```txt
Las reglas deciden que esta permitido.
La IA optimiza dentro de lo permitido.
Los validadores deciden que puede ejecutarse.
El executor ejecuta solo decisiones vigentes.
```
