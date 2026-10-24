<!--
Relatório de Impacto da Sincronização
- Versão: template sem versão → 1.0.0 (ratificação inicial)
- Fonte: DRAFT_CONSTITUITION.md (contexto curado de PRD, HLD, ADR-001 a 010, FDD-CORE-01 a 09
  e decisões do responsável de 2026-10-24)
- Princípios definidos (o template trazia 5 placeholders; o projeto adota 9):
  I. Fora do Caminho da Requisição (NÃO NEGOCIÁVEL)
  II. Segredo Nunca Sai (NÃO NEGOCIÁVEL)
  III. Sidecar como Fonte de Verdade
  IV. Contratos Evoluem só por Adição
  V. Memória Limitada e Degradação Explícita
  VI. Dono Único e Mínimo Privilégio
  VII. Plugin de Referência do SDK
  VIII. Teste Primeiro e Metas Medidas (NÃO NEGOCIÁVEL)
  IX. Simplicidade na v1
- Seções adicionadas: Restrições Técnicas e de Plataforma; Fluxo de Desenvolvimento e Portões de
  Qualidade; Governança
- Seções removidas: nenhuma
- Templates e documentos:
  ✅ .specify/templates/plan-template.md: Constitution Check com os portões I a IX e estrutura Go
  ✅ .specify/templates/tasks-template.md: testes obrigatórios (TDD), caminhos Go, tarefas de qualidade
  ✅ .specify/templates/spec-template.md: revisado, sem mudança necessária
  ✅ .specify/templates/checklist-template.md: revisado, sem mudança necessária
  ✅ README.md: link para a constituição e ordem de precedência atualizada
  ⚠ .specify/templates/commands/*.md: diretório inexistente, nada a revisar
  ⚠ specs/README.md: registrar que a spec 05 depende da 04 (decisão de 2026-10-24)
  ⚠ docs/ (HLD, PRD, FDDs e .json): textos superados listados em DRAFT_CONSTITUITION.md, seção 10
- TODOs adiados: nenhum
-->

# Constituição do n3x-plugin-core

O `n3x-plugin-core` (ID `n3x.core`, módulo `github.com/giustech/n3x-plugin-core`) é o plugin
obrigatório da plataforma N3X. A metade de proxy (`agent/`) roda no `libn3x.so` de cada sidecar Envoy e
faz login, registro, heartbeat, eventos e saída. A metade de controller (`controller/`) roda no binário do
controller, emite e valida o token de sessão, guarda em memória inventário, configs versionadas e
eventos, lê os usuários de leitura e monta as telas do painel. Esta constituição vale para todo código,
spec, plano e tarefa deste repositório.

## Princípios Fundamentais

### I. Fora do Caminho da Requisição (NÃO NEGOCIÁVEL)

- A metade de proxy NÃO DEVE declarar fase nem hook de requisição; o manifesto mantém `proxy.phases: []`.
- Todo trabalho da metade de proxy (login, registro, heartbeat, eventos e saída) DEVE rodar em tarefas
  `env.Background()`, usando no máximo 6 das tarefas permitidas por
  `runtime.plugin_limits["n3x.core"].background.max_tasks` (padrão 8).
- Nenhuma falha do core, nem a indisponibilidade do controller, PODE gerar erro, atraso ou bloqueio numa
  requisição da aplicação. Sem controller, o sidecar segue servindo tráfego.
- Toda chamada ao controller DEVE ter prazo e nova tentativa com backoff e jitter.

Justificativa: o orçamento do sidecar (p50 ≤ 0,5 ms e p99 ≤ 2 ms com o core ativo) não comporta
trabalho de controle no caminho quente, e o controller fica fora do caminho crítico por desenho (ADR-008).

### II. Segredo Nunca Sai (NÃO NEGOCIÁVEL)

- Nenhum token, segredo, nonce de sessão, hash de senha ou config sem máscara PODE aparecer em log, erro
  gRPC, métrica, evento, tela ou exportação.
- Segredo de plugin só sai do sidecar como marcador: `***:<12 hex>` (SHA-256 do texto `ENC[...]`, nunca do
  valor em claro), `***:plain` (com o evento `secret_in_clear`), `***:opaque` ou `***:omitted:<12 hex>`.
  Quando não consegue classificar uma config com segurança, o core DEVE falhar fechado e marcá-la como
  opaca.
- `session.hmac_key` DEVE vir selada em `ENC[...]`; em claro, a validação recusa e o controller não sobe.
- Nenhum pacote do core PODE importar `secret/` do SDK; segredos chegam como `plugin.Secret`.
- Todo pacote DEVE ter teste de canário de segredos, com `secrettest.Leaks` igual a 0.

Justificativa: um segredo exposto no painel ou num log é o risco de maior impacto do core. A impressão
digital calculada sobre o cifrado revela a troca de um segredo sem permitir ataque de dicionário (ADR-006).

### III. Sidecar como Fonte de Verdade

- O controller só observa: o core NÃO DEVE alterar a config de nenhum sidecar, e nenhuma tela do core
  tem formulário ou ação de escrita.
- Todo estado observado (inventário, manifesto, config e eventos) DEVE ser reconstruível pelos sidecars
  depois de um restart do controller, por novo login ou por `resend_register`, com volta ao painel em até
  30 s.
- O hash reportado DEVE corresponder à config em uso. O controller recalcula o hash e, se não bater,
  recusa com `DATA_LOSS` e mantém a última versão válida com alerta.
- A identidade de toda chamada ao serviço `Core` DEVE vir do token de sessão; identidade diferente recebe
  `PERMISSION_DENIED` e gera log de alerta.

Justificativa: a config de cada serviço é estática e só muda com redeploy, e o controller fica fora do
caminho crítico. O core espelha o estado dos sidecars, não o comanda (ADR-001, ADR-005).

### IV. Contratos Evoluem só por Adição

- O proto `n3x.plugins.n3x.core.v1` DEVE evoluir só por adição: campo novo sobe o `minor`; remoção,
  renomeação ou troca de tipo é barrada por `buf breaking`, bloqueante em todo PR a partir da primeira tag
  estável.
- A v1 aceita só a major 1. Outra major, ou `Register` sem `proto_version`, recebe `FAILED_PRECONDITION`.
- O formato da config efetiva (`format`) e o schema da seção `n3x.core` DEVEM crescer só por adição. Uma
  diferença que vem só do `format` não conta como divergência de config.
- Superfícies publicadas (superfície golden do proto, schemas golden e golden da config efetiva) são
  imutáveis; mudá-las exige nova versão declarada.
- Quando o formato do manifesto ou do proto muda, o controller DEVE ser atualizado antes dos sidecars.

Justificativa: sidecars antigos e controller novo convivem em todo rollout, e uma quebra deixaria
sidecars fora do painel (ADR-007).

### V. Memória Limitada e Degradação Explícita

- Todo dado guardado pelo core DEVE ter teto. O Store usa o `memory.Map` do SDK sob o `memory_limit`
  (16 MiB a 1 GiB, padrão 128 MiB), e a cota `runtime.plugin_limits["n3x.core"].memory.max_bytes` DEVE
  ser maior ou igual ao `memory_limit`; se for menor, a subida é recusada.
- Retenções: sidecar desconectado até `inventory_retention` (1 min a 24 h, padrão 24 h, sem fase de
  histórico); até 20 versões de config por serviço (teto mole: uma versão em uso por instância viva nunca
  sai); até 200 eventos por instância.
- O descarte DEVE ser explícito, feito pela tarefa `store_reclaim` a partir de 90% do teto, na ordem do
  ADR-009: desconectados mais antigos, eventos mais antigos e versões sem instância. Descarte silencioso
  pelo SDK é proibido.
- Se o uso continuar acima de 100% depois de um passe, o core DEVE entrar em estado degradado (banner no
  painel e `n3x_n3x_core_degraded = 1`), continuar aceitando heartbeats, eventos e mudanças de estado,
  recusar só config nova com `RESOURCE_EXHAUSTED` e sair do estado sozinho abaixo de 90%.

Justificativa: com tudo em memória, o teto explícito protege o orçamento do controller (150 MB e
0,15 vCPU com 500 sidecars) e preserva o que mais importa ao operador, que é saber quem está conectado
(ADR-009).

### VI. Dono Único e Mínimo Privilégio

- As capacidades privilegiadas (`PublicGRPC`, `Authenticate`, `CheckUser` e `ConnInterceptors`) pertencem
  só ao `n3x.core`, e `n3x.plugins.n3x.core.v1.Login/Login` é a única rota pública.
- O token DEVE ser HMAC-SHA256 com nonce, amarrado a serviço e instância, com TTL padrão de 1 h (5 min a
  24 h), renovação em 80% do prazo com jitter de 10% e validação em tempo constante (`hmac.Equal`). A
  chave HMAC pertence ao core e o token nunca fica visível a outro plugin.
- Usuários de leitura vêm só do `n3x.yaml` do controller (até 100, bcrypt de custo 12 a 16). O diretório é
  só leitura, e `ChangePassword` devolve "não suportado".
- Só o papel `sidecar` chama o serviço `Core`. As telas de sidecars, configs e fluxo exigem o papel de
  leitura, e a de usuários exige root.
- LGPD: IP e nome de usuário ficam só em memória, no máximo 24 h depois da desconexão, visíveis só a
  usuários autenticados. Exportações não levam endereço nem IP.

Justificativa: um sidecar não pode envenenar o inventário de outro, e um dono único para a autenticação
evita ampliar a superfície privilegiada do SDK (ADR-003, ADR-004, ADR-005).

### VII. Plugin de Referência do SDK

- O core DEVE usar só APIs públicas do `n3x-sdk`, nunca pacotes internos do controller ou do sidecar.
- Todo código DEVE passar na análise estática do SDK sem `//n3x:allow`: sem instrução `go` crua,
  `context.Background()`, `os.Exit`, `log.Fatal`, `init()` com efeito colateral, nem `unsafe` ou
  `reflect` sobre tipos do SDK.
- Regras de import, verificadas por teste com `go list -deps`: `effconfig` é puro (sem I/O, log,
  goroutine, gRPC ou SDK de controller); `types/` usa só a biblioteca padrão e `n3x-sdk/plugin`; `agent/`
  não importa `n3x-sdk/controller`.
- As telas DEVEM ser árvores de widgets do SDK; o core não escreve HTML.
- Métricas DEVEM ser declaradas só no catálogo `internal/metricdefs`, com o prefixo `n3x_n3x_core_`. Logs
  saem em JSON por `log/slog`, com ID do plugin, serviço e instância.

Justificativa: o core é o primeiro plugin de duas metades e o exemplo que os outros autores vão seguir;
ele não pode depender de nada que um plugin comum não tenha.

### VIII. Teste Primeiro e Metas Medidas (NÃO NEGOCIÁVEL)

- TDD é obrigatório: todo contrato e todo critério de aceite das FDDs vira teste, que DEVE falhar antes da
  implementação.
- Cobertura mínima de 90% de linhas em todos os pacotes.
- Toda suíte roda com `go test -race`. Parsers e handlers têm fuzzing (5 min por alvo na CI noturna).
  Caminhos com meta de zero alocação (`Authenticate`, `Touch` e heartbeat) têm benchmark `-benchmem` que
  falha se houver alocação.
- Toda meta numérica das FDDs (latência, memória, tempo de reconstrução) DEVE ter teste, benchmark ou
  cenário de carga que a meça.
- O core DEVE passar nas suítes de conformidade do SDK (`hostconformance` e `conformance.RunCore`).

Justificativa: o core entra em todo build, então um defeito dele atinge todos os serviços. Testes escritos
antes da implementação são a garantia de que os contratos das FDDs foram cumpridos.

### IX. Simplicidade na v1

- A v1 NÃO DEVE ter banco, disco, auditoria, tracing distribuído, gestão de usuários pelo painel, rotação
  de chave sem restart nem revogação individual de token. O controller roda com uma réplica.
- Todo armazenamento DEVE passar pela porta `Store`, e um adaptador futuro DEVE passar na mesma suíte
  `storetest`.
- Trazer para a v1 qualquer item excluído acima exige emenda desta constituição e um ADR.

Justificativa: o escopo enxuto é a mitigação do risco de capacidade (uma pessoa mantém cinco
repositórios), e a porta `Store` permite que a v2 troque só o adaptador (ADR-001, ADR-002).

## Restrições Técnicas e de Plataforma

- **Linguagem:** Go 1.26 ou superior, sem cgo, com `log/slog`; grpc-go e protobuf nas versões do `go.mod`
  do `n3x-sdk`; `golang.org/x/crypto/bcrypt` só para conferir hashes; JSON canônico (RFC 8785)
  implementado no próprio core, sem dependência externa.
- **Proto:** buf CLI 1.x (config `v2`, lint `STANDARD`, breaking `FILE`); `protoc-gen-go` e
  `protoc-gen-go-grpc` fixados no módulo separado `tools/`.
- **SDK:** `n3x-sdk` em pré-release `v0.x` durante o desenvolvimento. O `go.mod` usa o limite inferior da
  faixa `sdk` do manifesto, que exige `>=1.0.0 <2.0.0` na release.
- **Identidade:** manifesto `n3x-plugin.yaml` com `id: n3x.core`, duas metades e
  `controller.required: true`.
- **Distribuição:** sem artefato próprio. O `n3x-builder` embute o core no `libn3x.so` (alvos `envoy-1.40`
  e `istio-1.31`, em amd64 e arm64) e no binário do controller; remover o core falha o build (`N3X-B008`).
- **Ambientes:** Kubernetes, ECS e docker compose, sem recurso exclusivo de nenhum orquestrador; Istio só
  como compatibilidade. O controller roda com uma réplica.
- **Metas:** custo zero no caminho da requisição; 500 sidecars em 150 MB e 0,15 vCPU; painel com
  p95 < 300 ms; `Login` com p95 < 300 ms; `Register` com p95 < 20 ms; ack de heartbeat < 50 ms; volta ao
  painel em até 30 s; controller com 99,5% de disponibilidade ao mês.
- **Licença:** Apache-2.0.

## Fluxo de Desenvolvimento e Portões de Qualidade

- **Specs:** o código nasce pelas 10 specs de `specs/README.md`, na ordem de dependência. A spec 04 entrega
  só `validateSession`, e a spec 05 depende dela para compor `directory.ValidateUsers`. Cada spec passa por
  `specify`, `clarify`, `plan`, `tasks` e `implement`.
- **Idioma:** spec, plano, tarefas e documentação gerada em português do Brasil; nomes de arquivos,
  branches e identificadores em inglês.
- **Portões de CI em todo PR:** `buf lint`; `buf breaking`; `make generate` seguido de
  `git diff --exit-code`; `go vet`; `go test -race` com cobertura de no mínimo 90%; análise estática do
  SDK; canário de segredos; testes de dependência com `go list -deps`. Na CI noturna: fuzzing. Nas telas:
  axe (WCAG 2.1 AA). Nas regras de alerta: `promtool`.
- **Alertas da v1:** só os 5 alertas do HLD e 2 indicadores de SLO.
- **Precedência das fontes quando discordam:** esta constituição > blocos "Decisões aprovadas" das FDDs >
  corpo das FDDs > ADRs (o ADR-001 revisto conta como 2026-10-22) > HLD > PRD; o `.md` vence o `.json`. As
  divergências já resolvidas estão na seção 10 de `DRAFT_CONSTITUITION.md`.
- **Revisão:** todo PR DEVE citar a FDD e os critérios de aceite que atende e passar nos portões acima.
  Uma violação de princípio só entra com justificativa na tabela "Complexity Tracking" do plano e com
  aprovação do responsável pelo projeto.

## Governança

- Esta constituição prevalece sobre qualquer outra prática, documento ou template deste repositório.
- **Emendas:** por PR que altera este arquivo, com o Relatório de Impacto atualizado e os templates de
  `.specify/templates/` sincronizados. Mudança de decisão arquitetural exige também um ADR novo ou revisto
  em `docs/adr/`. Quem aprova é o responsável pelo projeto.
- **Versionamento semântico:** MAJOR para remoção ou redefinição incompatível de princípio ou de regra de
  governança; MINOR para princípio ou seção nova, ou para orientação ampliada de forma relevante; PATCH
  para esclarecimento e ajuste de redação.
- **Conformidade:** o "Constitution Check" do `/speckit-plan` DEVE verificar os princípios I a IX antes da
  pesquisa e de novo depois do design. O `/speckit-analyze` trata qualquer conflito com esta constituição
  como crítico.
- **Orientação de execução:** `CLAUDE.md` (regras do agente e idioma) e `README.md` (visão do projeto).

**Versão**: 1.0.0 | **Ratificada**: 2026-10-24 | **Última emenda**: 2026-10-24
