# Safe Automation Control Plane

Arquitectura open-source para automatizaciones seguras con IA: cuatro protocolos
complementarios que juntos forman una capa de control para plataformas que automatizan
acciones con impacto real.

La idea central:

```txt
Las reglas definen que esta permitido.
La IA optimiza dentro de lo permitido.
La plataforma valida antes de ejecutar.
El gateway outbound controla las llamadas al mundo externo.
```

---

## Los cuatro protocolos

| Protocolo | Responsabilidad | Documento |
|---|---|---|
| **AI Routing Decision Engine** | Decide si una accion puede avanzar, con que riesgo y estrategia | [protocolo →](./protocol-decision-engine.md) |
| **AI Model Layer** | Define como se comporta la IA: que puede proponer, que no puede hacer, como se eligen modelos | [protocolo →](./protocol-ai-model-layer.md) |
| **Provider Outbound Gateway** | Ejecuta llamadas a APIs externas de forma idempotente, limitada y auditable | [protocolo →](./protocol-provider-gateway.md) |
| **API Integration** | Receta paso a paso para agregar una nueva API externa al control plane | [protocolo →](./protocol-api-integration.md) |

---

## Flujo completo

### Decision Engine + AI Model Layer

```mermaid
flowchart TD
    A([Solicitud\nusuario · bot · asistente · worker · webhook]) --> B[Context Builder\nRoutingSnapshot con action.type]
    B --> C[Policy Engine\nplan · saldo · permisos · canal · opt-out · flags]

    C -->|blocked| Z1([block])
    C -->|allowed| D{Decision Cache}

    D -->|hit| E[Persistir cache hit\nen RouterDecision]
    E --> OUT

    D -->|miss| F[Provider Selector\nelige modelo por feature\ncosto · riesgo · plan · circuit breaker]

    F -->|sin candidatos| FB([ruleOnlyFallback])
    F -->|provider ok| G[AI Router\nllama al LLM]

    G -->|error / timeout| FB
    FB --> BV

    G -->|rawOutput| H[Schema Validator\nadditionalProperties: false]

    H -->|invalido — retry x1| G
    H -->|invalido x2| FB
    H -->|valido| IL[AI Usage Ledger\nregistra tokens reales]

    IL --> BV[Business Validator\nre-valida plan · saldo · permisos\ncanal · opt-out · expiracion]

    BV -->|falla regla dura| Z2([block])
    BV -->|requiresApproval| J[Approval Workflow]
    BV -->|ok| K[RouterDecision\nroutable · auditada · con expiresAt]

    J -->|aprobado| K
    J -->|rechazado| Z3([block])

    K --> L[EventOutbox\nauditoria atomica]
    L --> OUT([Executor\nsolo corre decisiones vigentes])

    style Z1 fill:#ef4444,color:#fff
    style Z2 fill:#ef4444,color:#fff
    style Z3 fill:#ef4444,color:#fff
    style FB fill:#f97316,color:#fff
    style K fill:#22c55e,color:#fff
    style OUT fill:#22c55e,color:#fff
```

### Provider Outbound Gateway

```mermaid
flowchart TD
    EX([Executor]) --> MF[Provider Registry\nresuelve manifest]
    MF --> GT[getValidToken]

    GT -->|token vivo| ID
    GT -->|vencido| RF[OAuth Refresher\nlease si token single-use]
    RF -->|ok| ID[Idempotencia\nprovider · idempotencyKey]
    RF -->|falla| TU([token_unavailable])

    ID -->|hit — ya ejecutado| RES([devuelve resultado\npersistido])
    ID -->|miss| CB[Circuit Breaker\npor provider · endpoint]

    CB -->|open| CO([circuit_open])
    CB -->|closed| RL[Rate Limit\npor clientId · provider]

    RL -->|excedido| RA([rate_limited])
    RL -->|ok| HTTP[HTTP call\ncon retry y backoff]

    HTTP -->|2xx| OK[ProviderHttpAttempt success\nApiCostLedger si aplica]
    HTTP -->|408 · 429 · 5xx| RT[backoff + reintentar\nhasta maxAttempts]
    HTTP -->|4xx permanente| ERR[ProviderHttpAttempt error\nno reintentar]
    RT --> HTTP

    OK --> API([API externa · respuesta])

    style TU fill:#ef4444,color:#fff
    style CO fill:#ef4444,color:#fff
    style RA fill:#ef4444,color:#fff
    style ERR fill:#ef4444,color:#fff
    style RES fill:#22c55e,color:#fff
    style API fill:#22c55e,color:#fff
```

### Webhooks entrantes

```mermaid
flowchart LR
    P([Provider externo]) --> W[POST /webhooks/:slug]
    W --> GW[webhookGateway\nvalida firma · deduplica\npersiste RawProviderEvent]
    GW -->|200 en menos de 500ms| P
    GW --> WK[worker async\nre-fetch via providerHttpGateway]
    WK --> CB[Context Builder\nRoutingSnapshot]
    CB --> DE([Decision Engine\nflujo completo])
```

El **Decision Engine** decide qué está permitido.
El **AI Model Layer** razona dentro del slot que el Decision Engine le asigna — nunca toca APIs externas.
El **Provider Gateway** ejecuta la llamada al mundo externo con control total.
El protocolo de **API Integration** explica cómo conectar una nueva API a esta cadena.

---

## La regla global

Nunca:

```txt
IA -> Executor directo
IA -> API externa directo
Módulo de negocio -> axios/fetch directo -> Provider externo
Asistente interno -> API externa directo
```

Siempre:

```txt
Acción → Decision Engine → Executor → Provider Gateway → API externa
```

---

## Flujo de implementacion recomendado

### Primer corte — seguridad antes que inteligencia

- Policy Engine con reglas duras.
- `RouterDecision` model + schema.
- `ruleOnlyFallback` — el motor opera sin IA desde el día uno.
- `ProviderAccount` + Provider Registry + manifests.
- `ProviderHttpAttempt` + `providerHttpGateway`.
- Endpoint de simulacion.

Resultado: la plataforma puede bloquear o permitir de forma conservadora,
sin depender de IA ni de ejecucion automatica.

### Segundo corte — IA controlada

- AI Router + Provider Selector.
- Decision Cache.
- AI Usage Ledger + Circuit Breaker.
- Schema Validator estricto.
- Business Validator completo.
- Approval Workflow.

Resultado: la IA recomienda, pero las reglas y validadores siguen teniendo la ultima palabra.

### Tercer corte — llamadas externas controladas

- Token refresh provider-specific con lease si el provider rota tokens.
- Rate limits por tenant/provider.
- Circuit breakers.
- Cost ledger externo.
- Workers idempotentes con DLQ.
- Modo advisory → enforced por canal estable.

Resultado: el executor llama providers externos sin perder control operativo.

---

## Modos de rollout

| Modo | Comportamiento | Cuándo usar |
|---|---|---|
| `simulation` | Calcula riesgo y costo, no ejecuta | Development, QA |
| `shadow` | Llama al modelo, registra, no afecta producción | Validar calidad antes de advisory |
| `advisory` | Sugiere, humano aprueba, executor espera | Rollout inicial, features nuevas |
| `enforced` | Puede ejecutar sin aprobación si validaciones pasan | Solo features maduras con tests |

Regla: canales experimentales o nuevos arrancan en `shadow`. Nunca `enforced` desde el primer día.

---

## Repos de cada protocolo

- [AI-Routing-Decision-Engine](https://github.com/cristiandkzk/AI-Routing-Decision-Engine) — motor de decisiones con tests y simulacion
- [Provider-Outbound-Gateway](https://github.com/cristiandkzk/Provider-Outbound-Gateway) — documentacion del gateway
- [Safe-Automation-Control-Plane](https://github.com/cristiandkzk/Safe-Automation-Control-Plane) — este repo (indice de protocolos)

---

## Licencia

MIT.

---

## Regla final

```txt
Las reglas deciden que esta permitido.
La IA optimiza dentro de lo permitido.
Los validadores deciden que puede ejecutarse.
El executor ejecuta solo decisiones vigentes.
El gateway outbound controla toda llamada al mundo externo.

Quien origina la decision no importa:
  un usuario, un bot, un asistente IA o un worker
  siempre pasan por el mismo motor.
```
