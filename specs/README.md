# Specs do n3x-plugin-core

Índice das pastas de especificação e das FDDs de cada uma. A numeração segue a ordem planejada de execução, respeitando as dependências. Cada pasta é uma sessão por fase do Spec Kit (`specify`, `clarify`, `plan`, `tasks`, `implement`).

As FDDs ficam em [`../docs/fdd/`](../docs/fdd/). As decisões aprovadas em cada FDD estão nos blocos "Decisões aprovadas" perto do início do documento, e prevalecem sobre o texto original quando houver divergência.

| Pasta | FDD | Depende de | Ressalva |
| --- | --- | --- | --- |
| [`01_fundacao_proto_manifesto_types`](01_fundacao_proto_manifesto_types) | [FDD-CORE-01](../docs/fdd/FDD-CORE-01-fundacao-proto-manifesto-e-types.md) | nenhuma | |
| [`02_catalogo_metricas`](02_catalogo_metricas) | [FDD-CORE-09](../docs/fdd/FDD-CORE-09-telas-cartoes-e-metricas.md), contrato 12 (catálogo e registro das métricas) | 01 | |
| [`03_config_efetiva_canonica_hash`](03_config_efetiva_canonica_hash) | [FDD-CORE-02](../docs/fdd/FDD-CORE-02-config-efetiva-canonica-e-hash.md) | 01, 02 | |
| [`04_login_sessao_hmac`](04_login_sessao_hmac) | [FDD-CORE-03](../docs/fdd/FDD-CORE-03-login-e-sessao-hmac.md) | 01, 02 | |
| [`05_diretorio_usuarios_leitura`](05_diretorio_usuarios_leitura) | [FDD-CORE-04](../docs/fdd/FDD-CORE-04-diretorio-de-usuarios.md) | 01, 02; melhor depois da 04 | |
| [`06_store_em_memoria`](06_store_em_memoria) | [FDD-CORE-05](../docs/fdd/FDD-CORE-05-store-em-memoria.md) | 01, 02; API do `memory.Map` do `n3x-sdk` | Contratos 3 a 5 ainda descrevem estruturas próprias; revisar para `memory.Map`. Confirmar a API do SDK na fase `plan`. |
| [`07_agente_registro_heartbeat`](07_agente_registro_heartbeat) | [FDD-CORE-06](../docs/fdd/FDD-CORE-06-agente-registro-heartbeat-e-eventos.md) | 01, 02, 03, 04 | |
| [`08_registry_inventario`](08_registry_inventario) | [FDD-CORE-07](../docs/fdd/FDD-CORE-07-registry-e-inventario.md) | 01 a 04, 06, 07 | Texto ainda cita o estado `history`, removido pela D-01 da FDD-07. |
| [`09_configs_versionadas_divergencia`](09_configs_versionadas_divergencia) | [FDD-CORE-08](../docs/fdd/FDD-CORE-08-configs-versionadas-e-divergencia.md) | 03, 06, 08 | Idem: referências a `history`. |
| [`10_telas_cartoes_alertas`](10_telas_cartoes_alertas) | [FDD-CORE-09](../docs/fdd/FDD-CORE-09-telas-cartoes-e-metricas.md), telas, cartões e alertas | 02, 05, 06, 08, 09 | Idem: filtros e telas que citam `history`. Só os 5 alertas do HLD e 2 indicadores de SLO entram na v1. |

A FDD-CORE-09 aparece em duas pastas de propósito, dividida por contrato: o catálogo de métricas precisa existir antes dos grupos 03 a 09, e as telas só podem vir depois deles.

## Documentos de referência

- PRD: [`../docs/prd/PRD-N3X.md`](../docs/prd/PRD-N3X.md)
- HLD: [`../docs/HLD.md`](../docs/HLD.md)
- ADRs: [`../docs/adr/`](../docs/adr/)
