# ADR-010: Métricas `n3x_n3x_core_*` e sem tracing na v1

Status: Aceito
Data: 2026-10-20

## Contexto
A ADR-022 do SDK define que as métricas de plugin usam o prefixo `n3x_<namespace>_<nome>_`. Já o HLD de plataforma citava métricas de negócio do core com o prefixo `n3x_controller_*`. As chamadas do core são de controle e ficam fora do caminho da requisição.

## Decisão
- **Métricas de negócio do core:** usam o prefixo `n3x_n3x_core_*`:
  - `sidecars{state}`
  - `divergences{service,kind}`
  - `identity_conflicts`
  - `registers_total{result}`
  - `logins_total{result}`
  - `events_total{kind}`
  - `memory_bytes`
  - `degraded`
  - `heartbeat_lag_seconds`
- **Catálogo completo:** o conjunto acima é o núcleo. O catálogo único das famílias `n3x_n3x_core_*` (35 da metade de controller e 12 da de proxy, entre elas `token_rejections_total`, `hmac_key_configured`, `session_index_entries`, `session_invalidations_total`, `events_rejected_total`, `heartbeat_streams`, `config_versions` e `panel_build_seconds`) fica em `internal/metricdefs`, especificado na FDD CORE-09.
- **Métricas do sidecar:** as que já existem (`n3x_controller_*`, definidas no HLD do sidecar) continuam como estão, alimentadas pelo core.
- **Tracing:** não há tracing distribuído na v1. A correlação é feita pelo `x-request-id` e pelo par serviço e instância nos logs.

## Justificativa
- Seguir a regra geral do SDK mantém o core como referência para os outros plugins.
- O tracing de chamadas de controle teria pouco valor diante do custo.

## Trade-off
- O HLD de plataforma precisa ser corrigido para o novo prefixo.
- Diagnosticar ponta a ponta depende de correlacionar logs.

## Requisitos relacionados
FR-006, FR-016; observabilidade operacional.
