# ADR-007: Proto `Core` com heartbeat bidirecional e `proto_version`

Status: Aceito
Data: 2026-10-20

## Contexto
O proto de registro, heartbeat, eventos e saída é contrato deste repositório. O FR-016 exige que o controller aceite a mesma major e recuse as outras com `FAILED_PRECONDITION`. O controller precisa conseguir pedir o reenvio do registro depois de perder o estado. Os eventos podem ser reenviados pelo buffer na reconexão.

## Decisão
- **Dois serviços no pacote `n3x.plugins.n3x.core.v1`:** `Login`, serviço próprio com o único método `Login` (rota pública), e `Core`, com:
  - `Register`
  - `Heartbeat`, como stream bidirecional
  - `Event`
  - `Exit`
- **Campos do `Register`:**
  - `proto_version{major, minor}`
  - identidade (serviço, instância e endereço)
  - alvo e versão do Envoy
  - manifesto e config (bytes JSON)
  - `config_hash` e `heartbeat_interval`
- **`RegisterResponse`:** leva `proto_version`, a versão do controller.
- **`HeartbeatAck`:** tem `resend_register` para quando o controller não conhece a instância.
- **`Event`:** tem `id` gerado no sidecar, e o controller ignora os repetidos. Os tipos são `EVENT_KIND_PANIC`, `EVENT_KIND_BREAKER_OPEN`, `EVENT_KIND_PLUGIN_DISABLED`, `EVENT_KIND_CONFIG_ERROR`, `EVENT_KIND_DRAIN_START` e `EVENT_KIND_SECRET_IN_CLEAR` (mais `EVENT_KIND_UNSPECIFIED = 0`, recusado). Também leva `plugin_instance` (6) e `suppressed` (7).
- **`Exit`:** é de melhor esforço, dentro dos 2 s do desligamento, e marca a instância como desconectada na hora.
- **Erros do core:**

  | Código | Quando |
  | --- | --- |
  | `INVALID_ARGUMENT` | Campos ou identidade ausentes |
  | `FAILED_PRECONDITION` | Major diferente |
  | `DATA_LOSS` | Hash não bate |
  | `PERMISSION_DENIED` | Identidade diferente da do token |
  | `RESOURCE_EXHAUSTED` | Teto de memória |

- **Desconexão:** o intervalo do heartbeat vem do novo campo `controller.heartbeat_interval` do `n3x.yaml` do sidecar, com padrão de 10 s. A instância fica desconectada depois de 3 intervalos sem ping.
- **Prazos:** o teto de 10 s por chamada do controller vale só para as unárias. O stream tem limite de 1 s por mensagem processada.
- **Evolução:** só por adição, com `buf breaking`. Um campo novo sobe o `minor`.

## Justificativa
- O stream bidirecional dá ao controller um canal de volta (o `resend_register`) sem precisar de uma chamada do controller para o sidecar.
- O `id` de evento torna o reenvio do buffer seguro.
- O intervalo por sidecar permite ajustar a carga por serviço.

## Trade-off
- O stream longo ocupa 1 dos 16 streams por conexão e exige a exceção de prazo no host.
- O `proto_version` repete uma informação que já está no nome do pacote. A redundância é aceita para dar uma recusa explícita.

## Requisitos relacionados
FR-002, FR-005, FR-006, FR-016; compatibilidade.
