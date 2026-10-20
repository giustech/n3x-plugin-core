# ADRs do n3x-plugin-core

Decisões de arquitetura registradas no HLD do plugin core (`../HLD.md`). A numeração é própria deste repositório. ADRs de outros repositórios são citados com o nome dele, por exemplo "ADR-019 do `n3x-sdk`". Cada ADR traz contexto, decisão, justificativa, trade-off e os requisitos relacionados do PRD (`../prd/PRD-N3X.md`).

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-armazenamento-em-memoria.md) | Armazenamento só em memória na v1, atrás da porta `Store` | Aceito |
| [ADR-002](ADR-002-auditoria-na-v2.md) | Auditoria adiada para a v2 | Aceito |
| [ADR-003](ADR-003-usuarios-pelo-yaml.md) | Usuários de leitura só pelo `n3x.yaml` | Aceito |
| [ADR-004](ADR-004-chave-hmac-do-core.md) | Chave HMAC do core, aleatória ou `session.hmac_key` | Aceito |
| [ADR-005](ADR-005-token-de-sessao.md) | Token de sessão com nonce, TTL de 1 h e identidade amarrada | Aceito |
| [ADR-006](ADR-006-config-efetiva-e-hash.md) | Config efetiva em JSON canônico, impressão digital de segredos e hash SHA-256 | Aceito |
| [ADR-007](ADR-007-proto-core.md) | Proto `Core` com heartbeat bidirecional e `proto_version` | Aceito |
| [ADR-008](ADR-008-sem-hooks-de-requisicao.md) | Core sem hooks de requisição | Aceito |
| [ADR-009](ADR-009-teto-de-memoria.md) | Teto de memória, ordem de descarte e estado degradado | Aceito |
| [ADR-010](ADR-010-metricas-e-tracing.md) | Métricas `n3x_n3x_core_*` e sem tracing na v1 | Aceito |
