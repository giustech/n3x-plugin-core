# DRAFT_CONSTITUITION — n3x-plugin-core

> Contexto curado para gerar a constituição com `/speckit-constitution`. Não é a constituição.
> Fontes: `docs/prd/PRD-N3X.md` (v1.5), `docs/HLD.md` (v1.0), `docs/adr/` (ADR-001 a 010),
> `docs/fdd/` (FDD-CORE-01 a 09), `specs/README.md` e decisões do responsável de 2026-10-24.

## 1. O que é

O `n3x-plugin-core` (ID `n3x.core`, módulo `github.com/giustech/n3x-plugin-core`) é o plugin obrigatório da
plataforma N3X: entra em todo build e não pode ser removido (`N3X-B008`). É o plugin de referência com as
duas metades do SDK:

- **Metade de proxy (`agent/`)**, no `libn3x.so` do sidecar Envoy: login, interceptor de sessão na conexão
  gRPC compartilhada, registro, heartbeat, eventos e saída. Não declara fase nem hook de requisição.
- **Metade de controller (`controller/`)**, no binário do controller: emite e valida o token de sessão,
  guarda em memória inventário, configs versionadas e eventos, lê os usuários de leitura e monta as telas.

Go 1.26 ou superior, sem cgo, licença Apache-2.0. Hoje só há documentação; o código nasce pelas 10 specs
de `specs/README.md`.

## 2. Problema que resolve

- O controller do N3X não tem armazenamento: sem o core, o painel não teria inventário, configs nem
  usuários de leitura.
- As capacidades privilegiadas (interceptor de conexão, autenticador de sessão, rota pública de login e
  diretório de usuários) precisam de um dono único; o SDK só as aceita do ID `n3x.core` deste módulo.
- Sidecar e controller precisam de um formato único de registro e de config efetiva, para detectar
  divergência entre instâncias sem nunca expor segredo.

## 3. Quem utiliza e onde

- **Time de plataforma/SRE** (público principal): escreve a seção do core no `n3x.yaml` do controller,
  gera os hashes bcrypt e acompanha painel, métricas e alertas.
- **Root e usuários de leitura** do painel. Só o root vê a lista de usuários.
- **Times de produto**, de forma indireta: o core é implícito no sidecar. **Outros plugins** usam a conexão
  com o token anexado pelo core, sem enxergá-lo. **Autores de plugin** o usam como referência.
- **Distribuição:** sem artefato próprio. O `n3x-builder` embute o core no `libn3x.so`
  (`sidecar:<semver>-<alvo>`) e no binário do controller (`controller:<semver>`).
- **Ambientes:** alvos Envoy 1.40 e istio-proxy 1.31, em amd64 e arm64; Kubernetes, ECS e docker compose,
  sem recurso exclusivo; Istio só como compatibilidade (`EnvoyFilter` estático). Controller com uma
  réplica. Sidecar sem controller configurado (standalone): metade de proxy inativa.
- Mantido por uma pessoa (risco de capacidade registrado no PRD).

## 4. O que faz

| Capacidade | Pacotes | FDD |
| --- | --- | --- |
| Proto, manifesto, `types/`, limites e CI de proto | `proto/`, `types/`, `n3x-plugin.yaml` | CORE-01 |
| Config efetiva canônica (RFC 8785), máscara e hash | `effconfig`, `effconfig/jcs`, `agent/effective` | CORE-02 |
| Login e sessão HMAC | `internal/sessiontoken`, `controller/auth`, `agent/session` | CORE-03 |
| Diretório de usuários só leitura | `controller/directory` | CORE-04 |
| Store em memória com teto, descarte e degradado | `controller/store`, `controller/store/memory` | CORE-05 |
| Registro, heartbeat, eventos e saída (sidecar) | `agent/registrar`, `heartbeat`, `events`, `exit` | CORE-06 |
| Registry, inventário, conflito e janitor | `controller/registry`, `inventory`, `janitor` | CORE-07 |
| Configs versionadas, divergência, diff e trecho YAML | `controller/configs` | CORE-08 |
| Telas, cartões, catálogo de métricas e alertas | `controller/panel`, `internal/metricdefs` | CORE-09 |

**Fora da v1:** auditoria (ADR-002); gestão de usuários pelo painel e troca de senha (ADR-003); banco e
mais de uma réplica (ADR-001); tracing (ADR-010); hooks de requisição (ADR-008); edição de config pelo
painel; rotação de chave sem restart; revogação de token.

## 5. Stack e restrições técnicas

- Go 1.26+, `log/slog`; grpc-go e protobuf nas versões do `go.mod` do SDK; `x/crypto/bcrypt` só para
  conferir; JSON canônico próprio, sem dependência externa; buf CLI 1.x (`v2`, lint `STANDARD`,
  breaking `FILE`) com geradores fixados em `tools/go.mod`.
- `n3x-sdk` em pré-release `v0.x` durante o desenvolvimento; o manifesto exige `>=1.0.0 <2.0.0` na release.
- Painel: o core só monta árvores de widgets do SDK; o host renderiza (Echo v5 + templ + htmx).
- Imports: `effconfig` é puro; `agent/` não importa `n3x-sdk/controller`; ninguém importa `secret/` do SDK;
  análise estática do SDK sem `//n3x:allow`. Métricas só pelo catálogo `internal/metricdefs` (35 + 12).

## 6. Contratos que o repositório possui

- **Proto `n3x.plugins.n3x.core.v1`:** `Login/Login` (única rota pública) e `Core` (`Register`, `Heartbeat`
  bidirecional, `Event`, `Exit`). Só a major 1 na v1; outra major recebe `FAILED_PRECONDITION`. Evolução
  só por adição, barrada por `buf breaking`.
- **Token:** `v1.<kid>.<iat>.<exp>.<nonce>.<serviço>.<instância>.<MAC>`, HMAC-SHA256, TTL de 1 h,
  renovação em 80% com jitter de 10%, `hmac.Equal`. A identidade das chamadas `Core` vem do token.
- **Config efetiva formato 1:** JSON canônico; segredos como `***:<12 hex>` (sobre o `ENC[...]`),
  `***:plain`, `***:opaque` ou `***:omitted:<12 hex>`; hash `sha256:<hex>` recalculado no controller.
- **Seção `n3x.core` do `n3x.yaml` do controller:** `inventory_retention` (1 min a 24 h, padrão 24 h),
  `memory_limit` (16 MiB a 1 GiB, padrão 128 MiB), `session.ttl` (5 min a 24 h, padrão 1 h),
  `session.hmac_key` (só `ENC[...]`, opcional), `users[]` (até 100; bcrypt de custo 12 a 16).
- **No `n3x.yaml` do sidecar:** `controller.heartbeat_interval` (1 s a 5 min, padrão 10 s) e
  `runtime.plugin_limits["n3x.core"]` (`background.max_tasks` padrão 8, mínimo 6; `memory.max_bytes`).
- **Manifesto:** `id: n3x.core`, duas metades, `proxy.phases: []`, `controller.required: true`.
- **Telas:** `/plugins/n3x/core/{sidecars,configs,fluxo,usuarios}` e 3 cartões; exportação sem IP.

## 7. Metas não funcionais

- Custo zero no caminho da requisição; nenhum erro da aplicação causado pelo core ou pelo controller.
- Volta ao painel em até 30 s depois de um restart (cerca de 10 s com `hmac_key`; sem ela, até ~200 serviços).
- 500 sidecars em 150 MB e 0,15 vCPU; painel com p95 < 300 ms; controller com 99,5% ao mês.
- Login p95 < 300 ms; `Register` p95 < 20 ms; ack do heartbeat < 50 ms; `Authenticate` e `Touch` sem alocação.
- Retenções: desconectado até `inventory_retention`; 20 versões por serviço (teto mole); 200 eventos por
  instância. Acima de 90% do teto, descarte na ordem do ADR-009; acima de 100%, estado degradado, que só
  recusa config nova com `RESOURCE_EXHAUSTED`.
- `memory_limit` é o teto do Store, e a cota `runtime.plugin_limits["n3x.core"].memory.max_bytes` precisa
  ser maior ou igual a ele; se for menor, a subida é recusada.

## 8. Segurança e privacidade

- Nenhum token, segredo, hash de senha ou config sem máscara em log, erro, métrica ou evento.
- `session.hmac_key` só em `ENC[...]`; em claro, o controller não sobe. TLS 1.3 em toda conexão.
- Identidade diferente da do token recebe `PERMISSION_DENIED`.
- LGPD: IP e usuário só em memória, no máximo 24 h depois da desconexão; exportação sem IP. A falta de
  auditoria na v1 é lacuna aceita (ADR-002).

## 9. Qualidade, testes e fluxo

- TDD obrigatório: todo contrato e critério de aceite das FDDs vira teste, que falha antes da implementação.
- Cobertura mínima de 90% de linhas em todos os pacotes.
- CI em todo PR: `buf lint`, `buf breaking`, `make generate` + `git diff --exit-code`, `go vet`,
  `go test -race`. CI noturna: fuzzing de 5 min por alvo.
- Testes de tabela e de propriedade, goldens, benchmarks `-benchmem`, conformidade do SDK
  (`hostconformance`, `conformance.RunCore`), canário de segredos (`secrettest.Leaks` = 0), `promtool` nos
  alertas e axe (WCAG 2.1 AA) nas telas.
- Spec Kit em 10 specs ordenadas por dependência; documentos em PT-BR, identificadores em inglês.

## 10. Precedência das fontes e divergências resolvidas

Ordem: decisões deste draft > blocos "Decisões aprovadas" das FDDs > corpo das FDDs > ADRs (ADR-001 revisto
conta como 2026-10-22) > HLD > PRD. O `.md` vence o `.json` (os `.json` não têm os blocos de decisão).

- Não existe estado `history`: só `connected` e `disconnected` (D-01 da CORE-07). Os textos da CORE-05 a 09
  que citam `history` ou retenção de 48 h estão superados.
- `inventory_retention` vai até 24 h, por LGPD; a faixa de até 7 dias da D-03 da CORE-01 fica superada.
- O Store usa o `memory.Map` do SDK (D-01 da CORE-05); quem descarta é a tarefa `store_reclaim`, não o
  janitor; os usuários de leitura ficam fora do Store.
- Métricas nascem só no catálogo, mesmo onde a FDD cita `env.Metrics()` direto. Na v1, só os 5 alertas do
  HLD e 2 SLO (D-01 da CORE-09).
- Proto incompatível recebe `FAILED_PRECONDITION`, não o `UNIMPLEMENTED` do FR-005.
- O `Exit` sai só no desligamento, nunca no início da drenagem, e não desconecta se outra sessão estiver
  ativa. Conflito de identidade: dois streams com ping por 2 intervalos (D-03 da CORE-07).
- O token fica num `atomic.Pointer` do core, não em `env.Memory()`.
- Plugin sem schema tem a config enviada como `***:opaque` ("config não exibida"), não como YAML bruto.
- Standalone é o sidecar sem controller configurado; controller fora do ar não é standalone.
- A spec 04 entrega só `validateSession`; a composição com `directory.ValidateUsers` entra na spec 05.

## 11. Princípios candidatos (derivados dos ADRs e do HLD)

1. Fora do caminho da requisição: o core nunca afeta o tráfego (ADR-008).
2. Segredo nunca sai: só impressão digital do cifrado; nada sensível em log ou erro (ADR-006).
3. O sidecar é a fonte de verdade: o controller só observa e reconstrói o estado (ADR-001).
4. Contratos só crescem por adição: proto, formato da config efetiva e schema da seção (ADR-007).
5. Memória limitada e degradação explícita, nunca descarte silencioso (ADR-009).
6. Dono único e mínimo privilégio nas capacidades privilegiadas (ADR-004, ADR-005).
7. Plugin de referência: usa só APIs públicas do SDK e passa na análise estática dele.
8. Teste primeiro e metas numéricas medidas na CI.
9. Simplicidade na v1: sem banco, sem auditoria, uma réplica, porta `Store` pronta para a v2.
