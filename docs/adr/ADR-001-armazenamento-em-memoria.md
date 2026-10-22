# ADR-001: Armazenamento só em memória na v1, atrás da porta `Store`

Status: Aceito
Data: 2026-10-20

## Contexto
O controller não tem armazenamento próprio, e o PRD deixou para este HLD a definição do banco do core: tipo, modo em memória, retenção e comportamento degradado. Na v1, o controller roda com uma réplica só. Inventário, configs e eventos são estado observado, que os sidecars reenviam a cada login ou quando o controller pede.

## Decisão
- Na v1, o core guarda tudo em memória, sem banco e sem disco.
- O armazenamento fica atrás da porta `Store` (inventário, configs, eventos e usuários), com uma única implementação na v1, a `memory`.
- O adaptador usa o `memory.Map` do SDK e fica sujeito à cota do SDK (decisão revista em 2026-10-22; o texto original previa estruturas de Go do próprio core, fora da cota). Para manter o descarte explícito e o estado degradado do ADR-009, o core consulta o uso e a cota, remove itens de forma explícita e detecta o teto atingido, o que o `n3x-sdk` já expõe (confirmação do responsável; a conferência contra o código do SDK fica para a fase `plan`).
- O `memory_limit` (padrão 128 MiB) e a cota de memória do core no SDK (`runtime.plugin_limits["n3x.core"].memory.max_bytes`) precisam ser alinhados na implementação, já que a cota padrão do controller é menor (64 MiB).
- O campo `storage.dsn` sai da seção `plugins.n3x.core` do `n3x.yaml` do controller.
- Depois de um restart, o estado se reconstrói pelos sidecars, com novo login ou com o `resend_register` do heartbeat.

## Justificativa
- Não exige infraestrutura extra em nenhum ambiente (Kubernetes, ECS ou compose).
- É coerente com uma réplica só e com o sidecar como fonte de verdade.
- A porta `Store` deixa o banco da v2 trocar só o adaptador.

## Trade-off
- O histórico de hashes, de eventos e de sidecars desconectados se perde em cada restart.
- Inviabiliza a auditoria durável na v1 (ADR-002) e os usuários criados pelo painel (ADR-003).
- Não permite mais de uma réplica do controller.

## Requisitos relacionados
FR-006, FR-007, FR-009, FR-011, FR-016; disponibilidade e volta ao painel em até 30 s.
