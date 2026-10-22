### HLD: n3x-plugin-core (registro, sessão, inventário, configs e usuários de leitura)

Versão: 1.0
Data: 2026-10-20
Responsável: Guilherme Giusti

---

### Objetivo técnico
Ser o plugin obrigatório do N3X, presente em todo build, com duas metades:

- **Metade de proxy (dentro do `libn3x.so`):**
  - faz o login no controller e instala o interceptor de sessão na conexão gRPC compartilhada do sidecar
  - registra o sidecar: identidade, alvo, versão do Envoy, manifesto, e config efetiva mascarada com o hash dela
  - mantém o stream de heartbeat, envia os eventos e avisa a saída
- **Metade de controller:**
  - emite e valida o token de sessão HMAC
  - mantém em memória o inventário, as configs versionadas pelo hash, os eventos e os usuários de leitura
  - entrega ao painel as telas de sidecars, configs, fluxo e usuários, só com widgets do SDK

Ele também é o plugin de referência com as duas metades.

O core atende o FR-016 e as partes de dados do FR-006, FR-007 e FR-009, além da sessão do FR-002 e do FR-005. Na v1, tudo fica em memória e a auditoria (FR-010) fica para a v2 (ADR-001 e ADR-002).

**Problemas técnicos endereçados**
- O controller não tem armazenamento próprio. Sem o core, o painel não teria inventário, configs nem usuários de leitura.
- As capacidades privilegiadas (interceptor, autenticador, rota pública de login e diretório de usuários) precisam de um único dono confiável.
- Sidecar e controller precisam de um único formato de registro e de config efetiva, para que seja possível detectar divergência entre instâncias sem nunca expor segredo.

Este HLD é dono destes contratos (conforme `docs/hld/HLDs-N3X.md`):
- Proto `n3x.plugins.n3x.core.v1` (login, registro, heartbeat, eventos e saída) e a compatibilidade de versão entre sidecar e controller
- Formato da config efetiva, mascaramento de segredos e hash
- Token de sessão HMAC e a chave dele
- Armazenamento do core (na v1, em memória): inventário, configs, eventos e usuários de leitura, com retenções, teto de memória e estado degradado
- Seção `plugins.n3x.core` do `n3x.yaml` do controller

Dependências com outros sistemas
- `n3x-sdk`: contratos de plugin de proxy e de controller, capacidades privilegiadas (ADR-019 do SDK), `plugin.Secret`, widgets, memória isolada, padrão de IDs e de métricas e kit de testes. Esses contratos são só referenciados aqui.
- `n3x-sidecar`: conexão gRPC compartilhada (`core/grpcconn`) com o buffer de 1.000 itens, leitura do manifesto da seção ELF, canal de controle do entrypoint (drenagem e SIGTERM), identidade da instância e o campo `controller.heartbeat_interval` do `n3x.yaml` do serviço.
- `n3x-controller`: servidor gRPC com a ordem dos interceptors, host de plugins, sessão do painel, root, verificador de credencial (`env.Credentials()`) e log operacional do login do painel.
- `n3x-builder`: inclui o core sempre, na versão da imagem de build (`N3X-B008` se for removido), e embute o manifesto lido pelo core.

---

### Arquitetura geral
O core é um repositório Go que segue a estrutura padrão de plugin do SDK. A metade `agent/` é compilada dentro do `.so` do sidecar e a metade `controller/` dentro do binário do controller. As duas falam pelo proto `Core`, que trafega sobre a conexão TLS 1.3 já mantida pelo sidecar. Nenhum trabalho do core acontece no caminho da requisição (ADR-008): tudo roda em tarefas de fundo (`env.Background`).

```
 Sidecar (Envoy + libn3x.so)                         Controller (1 réplica)
 ┌──────────────────────────────────┐               ┌──────────────────────────────────────┐
 │ agent/ (metade de proxy)         │  TLS 1.3      │ host: interceptors → autenticação ───┐│
 │  session ── interceptor ─────────┼──────────────▶│                    (Authenticate) ◀──┘│
 │  registrar  heartbeat  events    │ n3x.plugins.  │ controller/ (metade de controller)   │
 │  exit                            │ n3x.core.v1   │  auth  registry  inventory  configs  │
 │        ▲ manifesto ELF, config   │   .Core       │  directory  janitor  panel  metrics  │
 │        ▲ canal de controle       │               │        │                             │
 └──────────────────────────────────┘               │        ▼                             │
                                                    │  Store (porta) → memory (v1)         │
                                                    └──────────────┬───────────────────────┘
                                                                   ▼
                                                     Painel (widgets): sidecars, configs,
                                                     fluxo, usuários e cartões
```

Ambiente de implantação
- Mesmo ambiente do controller e dos sidecars: Kubernetes, ECS ou docker compose, com Istio opcional
- O controller roda com uma réplica e o core não usa banco nem disco na v1. A escala é vertical, dentro do orçamento do HLD do controller (150 MB e 0,15 vCPU com 500 sidecars).
- Não existe artefato próprio: o core é embutido nos artefatos do sidecar e do controller pelo builder.

Tecnologias principais
- Go 1.26 ou superior, sem cgo, `log/slog`
- grpc-go e protobuf, com `buf generate` e `buf breaking` na CI
- `crypto/hmac` com SHA-256 e `crypto/rand`, e `golang.org/x/crypto/bcrypt` (só para conferir os hashes)
- JSON canônico pela RFC 8785, implementado no próprio core sem dependência externa (hipótese)
- `n3x-sdk`: `proxy.Define`, `ctrl.Define`, widgets, `memory`, `plugin.Secret` e kit de testes

Padrões adotados
- Plugin de duas metades conforme o SDK, com capacidades privilegiadas exclusivas do ID `n3x.core`
- Porta e adaptador no armazenamento: a interface `Store`, com a implementação `memory` na v1 e um banco na v2 (ADR-001)
- Stream gRPC bidirecional para o heartbeat; chamadas unárias para login, registro, eventos e saída (ADR-007)
- Estado observado com o sidecar como fonte de verdade, reconstruído depois de um restart do controller
- Snapshots imutáveis por serviço (copy-on-write) para a leitura do painel sem bloquear quem escreve
- Evolução do proto só por adição, com `proto_version` no registro

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependências |
| ----------- | ----------------- | ------------ |
| `agent/session` | Login com prazo de 10 s, guarda do token em `env.Memory()`, interceptor que injeta o token e impõe `min(ctx, 5 s)`, renovação em 80% do TTL com jitter de 10%, novo login em `UNAUTHENTICATED` | `core/grpcconn` do sidecar, `env.Memory()`, `env.Background()` e `controller.secret` |
| `agent/registrar` | Monta a identidade, lê o manifesto ELF, gera a config efetiva canônica, mascara os segredos pelo schema, calcula o hash e envia o `Register` (de novo após cada login e quando o ack pede `resend_register`) | `session`, schemas dos plugins e manifesto |
| `agent/heartbeat` | Abre o stream no intervalo de `controller.heartbeat_interval`, envia o ping, trata o ack e reabre o stream depois de uma queda, com backoff | `session` e `env.Background()` |
| `agent/events` | Recebe do core do sidecar os eventos de panic, disjuntor aberto, plugin desligado, erro de config, drenagem e segredo em claro, gera o `id` de idempotência e envia pelo buffer | Buffer da conexão compartilhada |
| `agent/exit` | No SIGTERM ou no início da drenagem, chama `Exit` em modo de melhor esforço, dentro dos 2 s | Canal de controle do entrypoint |
| `controller/auth` | Handler `Login` (rota pública, via `PublicGRPC`): confere a credencial com `env.Credentials().Verify` e emite o token. Fornece também o `Authenticate`, que valida o token para o interceptor do host. É dono da chave HMAC (ADR-004). | `env.Credentials()` e `session.hmac_key` |
| `controller/registry` | Serviço `Core`: valida os campos, a `proto_version`, o hash e se a identidade bate com a do token. Repassa para o `inventory` e o `configs`. | `auth`, `inventory` e `configs` |
| `controller/inventory` | Instâncias e sessões, estados (conectado e desconectado), conflito de identidade, divergência de manifesto e eventos | `Store` |
| `controller/configs` | Versões de config por hash, a config atual de cada instância, a divergência de config no serviço e o diff entre versões e entre instâncias | `Store` |
| `controller/directory` | Diretório de usuários de leitura vindos do `n3x.yaml`: `CheckUser` com bcrypt, e `ChangePassword` que devolve "não suportado" (ADR-003) | Config tipada |
| `controller/store/memory` | Implementação em memória da porta `Store`, com contabilidade de bytes, teto, ordem de descarte e snapshots imutáveis (ADR-009) | Nenhuma |
| `controller/janitor` | Marca como desconectado depois de 3 intervalos sem heartbeat, remove do inventário depois de `inventory_retention`, aplica as retenções e o descarte | `inventory`, `configs` e `env.Background()` |
| `controller/panel` | Telas com widgets (sidecars, configs, fluxo e usuários) e cartões da visão geral | `inventory`, `configs` e `directory` |
| `controller/metrics` | Métricas de negócio `n3x_n3x_core_*` | `env.Metrics()` |
| `proto/` e `types/` | `.proto` do serviço `Core` com os stubs gerados, e as structs de config das duas metades com tags `n3x` e `desc` | buf e SDK |

---

### Fluxo de requisições e de dados
**Fluxo de requisição**
- **Subida do sidecar.** O `session` cria o interceptor e chama `Login(serviço, instância, segredo)`, com prazo de 10 s.
- **Login no controller.**
  - O host aplica os limites por IP e por serviço. Se o chamador estiver bloqueado, devolve `RESOURCE_EXHAUSTED`.
  - O `auth` chama `env.Credentials().Verify`, que confere o bcrypt, com cache de 1 h e no máximo 2 bcrypt simultâneos.
  - Se a credencial for válida, o `auth` emite o token sobre serviço, instância, emitido em, expira em e nonce. O TTL é de 1 h.
- **Registro.**
  - O `registrar` envia o `Register`: `proto_version`, identidade, alvo, versão do Envoy, manifesto, config canônica mascarada, `config_hash` e `heartbeat_interval`.
  - O interceptor do host valida o token com o `Authenticate` e devolve `{papel: sidecar, serviço, instância, nonce}`.
  - O `registry` valida as condições abaixo, na ordem:

    | Condição | Erro |
    | --- | --- |
    | Major diferente de 1 | `FAILED_PRECONDITION` |
    | Campos ausentes | `INVALID_ARGUMENT` |
    | Identidade diferente da do token | `PERMISSION_DENIED`, mais um log de alerta |
    | Hash recalculado não bate | `DATA_LOSS` |

  - Com tudo válido, o inventário faz um upsert pela identidade, sem duplicar.
  - Se o hash for novo, o controller guarda a versão da config. Sem memória para isso, devolve `RESOURCE_EXHAUSTED`.
  - Por fim, o controller recalcula as divergências do serviço.
- **Heartbeat.**
  - O `heartbeat` abre o stream e envia um ping a cada intervalo. O controller atualiza o "visto por último" e responde com um ack em menos de 50 ms.
  - Se o controller não conhece a instância, por exemplo depois de um restart com a mesma chave, o ack leva `resend_register=true`.
- **Eventos.** São enviados por `Event` unário, com `id`, e passam pelo buffer de 1.000 itens quando o controller está fora do ar. O controller ignora `id` repetido.
- **Saída.** No SIGTERM, o `exit` chama `Exit(reason)` dentro dos 2 s, e o controller marca a instância como desconectada na hora.
- **Painel.** O host chama as telas do `panel` no polling de 5 s, e as telas leem os snapshots do `Store`.

**Fluxos alternativos**
- **Controller fora do ar:**
  - o sidecar continua servindo tráfego, em modo standalone
  - os eventos ficam no buffer, que descarta os mais antigos
  - o login é refeito com backoff de 1 a 60 s e jitter de 20%
- **Restart do controller com `session.hmac_key`:** os tokens continuam válidos. O primeiro heartbeat recebe `resend_register`, e o inventário se reconstrói em cerca de 1 intervalo.
- **Restart do controller sem `session.hmac_key`:** a chave nova invalida os tokens. O próximo envio recebe `UNAUTHENTICATED`, o sidecar refaz o login com jitter e reenvia o registro. Como o cache de credencial do host é por serviço, o custo é de cerca de um bcrypt por serviço (cerca de 12 s de CPU com 50 serviços).
- **Três intervalos sem heartbeat:** a instância fica desconectada. Depois de `inventory_retention` (24 h desde a desconexão), é removida do inventário; não há fase de histórico separada.
- **Conflito de identidade:**
  - acontece quando chega um segundo `Register` com a mesma identidade e outro nonce, e as duas sessões mandam heartbeat no mesmo intervalo
  - o controller marca o conflito no painel, no log e na métrica, e mantém as duas sessões
  - o conflito se resolve sozinho depois de 3 intervalos sem heartbeat da sessão antiga
- **Segredo em claro na config:** o sidecar envia `***:plain` e o evento `secret_in_clear`. O valor nunca sai do processo.

**Fluxo de dados**
- `n3x.yaml` do serviço e os defaults dos plugins → config efetiva no `.so` → JSON canônico (RFC 8785) → mascaramento pelo schema → hash `sha256:` → `Register` → recálculo do hash no controller → `ConfigVersion` deduplicada em memória → diff e divergência → telas do painel
- Manifesto ELF `.n3x_manifest` → `Register` → `Manifest` deduplicado pelo hash → comparação com o manifesto do binário do controller → alerta de plugins diferentes
- Core do sidecar (panic, disjuntor, desligamento, erro de config, drenagem) → `events` → buffer → `Event` → eventos da instância em memória (até 200) → detalhe do sidecar
- Heartbeat → "visto por último" → `janitor` → estado → cartões e métricas
- Seção `users` do `n3x.yaml` do controller → `Validate` → `directory` em memória → `CheckUser` no login do painel
- Nada é replicado nem persistido em disco na v1. A perda no restart é aceita e compensada pela reconstrução feita pelos sidecars.

---

### Modelo de dados (alto nível)
Entidades principais
- `Service`: nome do serviço, que agrupa as instâncias
- `Instance`:
  - serviço, instância e endereço
  - alvo (`envoy-1.40` ou `istio-1.31`) e versão do Envoy
  - estado (conectado ou desconectado) e "visto por último"
  - ponteiros para o manifesto e para a config atual
  - histórico de hashes
- `Session`: nonce, emitido em, expira em, intervalo do heartbeat e último ping
- `Manifest`: plugins, versões e metades compilados no `.so`, guardados pelo hash do próprio manifesto
- `ConfigVersion`:
  - hash `sha256:` e config efetiva mascarada em JSON canônico
  - visto pela primeira vez e visto pela última vez
  - deduplicada entre instâncias
- `Event`: `id`, instância, tipo, plugin, detalhe e horário
- `User`: nome, hash bcrypt e desativado, carregado do `n3x.yaml` na subida

**Formato da config efetiva, formato 1 (ADR-006)**

Exemplo de config efetiva já mascarada:
```json
{
  "format": 1,
  "sidecar": {
    "service": "api-pedidos",
    "listener": { "port": 17000 },
    "controller": { "address": "controller:17443", "secret": "***:9f2c41ab07de", "heartbeat_interval": "10s" }
  },
  "plugins": [
    { "id": "acme.ratelimit", "instance": "rl-api", "phase": "traffic", "order": 10, "failure": "open",
      "async": { "max_inflight": 1000, "timeout": "200ms" }, "match": { "grpc_services": [], "methods": [], "paths": [] },
      "config": { "limit": 100, "redis_password": "***:51e0c3aa9b12" } }
  ]
}
```
- O JSON é canônico pela RFC 8785: chaves ordenadas, números normalizados e sem espaços. O exemplo acima foi formatado só para leitura.
- **Mascaramento:** todo campo marcado como segredo no schema do plugin (tipo `plugin.Secret`), e também o `controller.secret` do sidecar, vira `***:` seguido da impressão digital. A impressão digital são os 12 primeiros caracteres hex do SHA-256 do texto `ENC[...]`, ou seja, do cifrado e nunca do valor em claro.
- **Segredo sem `ENC[...]`:** vira `***:plain`, e o sidecar emite o evento `secret_in_clear`.
- **Config de uma instância que o core não consegue mascarar com segurança** (sem schema, marcas incoerentes): `***:opaque`. **Documento acima de 512 KiB:** as maiores configs viram `***:omitted:<12 hex>` até caber. Segredo vazio fica `""`.
- **Inteiros fora de ±(2^53 − 1):** saem como texto com os dígitos exatos, um desvio explícito da RFC 8785 que o controller não precisa conhecer para conferir a forma canônica.
- **Ordem de `plugins[]`:** fase, depois `order`, depois `instance`; cada item leva também `async` e `match`.
- **Hash:** `sha256:<hex>` do JSON canônico já mascarado. Por isso, trocar um segredo muda o hash sem expor o valor.

Relações
- Um `Service` tem várias `Instance`.
- Uma `Instance` tem uma ou mais `Session`. Mais de uma sessão ativa ao mesmo tempo significa conflito de identidade.
- Uma `Instance` aponta para um `Manifest` e para a `ConfigVersion` atual, e guarda o histórico de hashes.
- Uma `ConfigVersion` é compartilhada por todas as instâncias com o mesmo hash.
- Uma `Instance` tem até 200 `Event`.
- `User` não tem relação com as demais entidades.

Fonte de verdade
- Estado observado (inventário, manifesto, config e eventos): o próprio sidecar. O core só espelha o estado em memória e o reconstrói depois de um restart.
- Usuários de leitura: a seção `plugins.n3x.core.users` do `n3x.yaml` do controller.
- Root: o `n3x.yaml` do host do controller, fora do core.
- **Retenção em memória (ADR-009):**
  - sidecar desconectado fica no máximo 24 h em memória (`inventory_retention`), sem fase de histórico
  - até 20 versões de config por serviço, descartando primeiro as que não têm instância apontando
  - até 200 eventos por instância
  - teto total de `memory_limit` (128 MiB)
- **Versionamento:** as configs são versionadas pelo hash. O formato da config tem o campo `format`, que só cresce por adição. A diferença que vem só do `format` (upgrade do core) não conta como divergência de config.
- **Teto de versões:** até 20 por serviço; uma versão em uso por instância viva nunca é descartada, então o serviço pode passar de 20.

---

### Interfaces públicas
| Nome | Tipo | Protocolo | Exposição | SLAs/Limites |
| ---- | ---- | ---------- | --------- | ------------- |
| `n3x.plugins.n3x.core.v1.Login/Login` | API | gRPC (protobuf), TLS 1.3, rota pública sem sessão | Interna (sidecar → controller) | p95 < 300 ms; limites de login do host por IP e serviço |
| `n3x.plugins.n3x.core.v1.Core/Register` | API | gRPC unário | Interna | Mensagem até 1 MiB; p95 < 20 ms |
| `n3x.plugins.n3x.core.v1.Core/Heartbeat` | Stream | gRPC bidirecional | Interna | Ack < 50 ms; 1 s por mensagem processada; sem teto de 10 s de chamada |
| `n3x.plugins.n3x.core.v1.Core/Event` | API | gRPC unário, idempotente por `id` | Interna | p95 < 20 ms |
| `n3x.plugins.n3x.core.v1.Core/Exit` | API | gRPC unário, melhor esforço | Interna | Dentro dos 2 s do desligamento |
| Capacidades privilegiadas (`PublicGRPC`, `Authenticate`, `CheckUser`, `ConnInterceptors`) | SDK | Interfaces Go do ADR-019 do SDK | Interna (host ↔ core) | `Authenticate` sem alocação no caminho comum; `CheckUser` dominado pelo bcrypt |
| Telas `/plugins/n3x/core/{sidecars,configs,fluxo,usuarios}` e cartões | SDK | Árvore de widgets renderizada pelo host (HTTPS) | Interna (painel) | Página até 10 s e cartão até 1 s (limites do SDK); meta de p95 < 300 ms com 500 sidecars |
| Métricas `n3x_n3x_core_*` | API | Prometheus, pelo endpoint do controller | Interna | Atualizadas a cada mudança de estado |

**Proto (ADR-007)**
```proto
syntax = "proto3";
package n3x.plugins.n3x.core.v1;

service Login {
  rpc Login(LoginRequest) returns (LoginResponse);   // rota pública, serviço próprio (FDD CORE-01, D-01)
}

service Core {
  rpc Register(RegisterRequest) returns (RegisterResponse);
  rpc Heartbeat(stream HeartbeatPing) returns (stream HeartbeatAck);
  rpc Event(EventRequest) returns (EventResponse);
  rpc Exit(ExitRequest) returns (ExitResponse);
}

message LoginRequest   { string service = 1; string instance = 2; string secret = 3; }
message LoginResponse  { string token = 1; google.protobuf.Timestamp expires_at = 2; }
message ProtoVersion   { uint32 major = 1; uint32 minor = 2; }
message Identity       { string service = 1; string instance = 2; string address = 3; }
message RegisterRequest {
  ProtoVersion proto_version = 1; Identity identity = 2; string target = 3;
  string envoy_version = 4; bytes manifest = 5; bytes config = 6;
  string config_hash = 7; google.protobuf.Duration heartbeat_interval = 8;
}
message RegisterResponse { ProtoVersion proto_version = 1; }   // versão do controller
message HeartbeatPing  { uint64 seq = 1; google.protobuf.Timestamp sent_at = 2; }
message HeartbeatAck   { uint64 seq = 1; bool resend_register = 2; }
enum EventKind {
  EVENT_KIND_UNSPECIFIED = 0; EVENT_KIND_PANIC = 1; EVENT_KIND_BREAKER_OPEN = 2;
  EVENT_KIND_PLUGIN_DISABLED = 3; EVENT_KIND_CONFIG_ERROR = 4; EVENT_KIND_DRAIN_START = 5;
  EVENT_KIND_SECRET_IN_CLEAR = 6;
}
message EventRequest {
  string id = 1; EventKind kind = 2; string plugin = 3; string detail = 4;
  google.protobuf.Timestamp occurred_at = 5;
  string plugin_instance = 6; uint32 suppressed = 7;
}
message EventResponse { }
message ExitRequest   { string reason = 1; }   // sigterm | drain
message ExitResponse  { }
```
- **Erros definidos pelo core:**

  | Código | Quando |
  | --- | --- |
  | `INVALID_ARGUMENT` | Campos ou identidade ausentes |
  | `FAILED_PRECONDITION` | Major diferente da do controller |
  | `DATA_LOSS` | Hash não bate com a config |
  | `PERMISSION_DENIED` | Identidade diferente da do token |
  | `RESOURCE_EXHAUSTED` | Teto de memória atingido ao receber uma nova versão de config |

  `UNAUTHENTICATED` e os demais códigos seguem o HLD do controller.
- **Compatibilidade:**
  - o controller aceita qualquer `minor` da mesma major
  - um campo novo sobe o `minor`
  - a remoção ou mudança de tipo é barrada pelo `buf breaking`

**Seção do core no `n3x.yaml` do controller (contrato deste HLD)**
```yaml
plugins:
  n3x.core:
    inventory_retention: 24h      # tempo total em memória após a desconexão
    memory_limit: 128MiB          # teto do armazenamento em memória
    session:
      ttl: 1h
      hmac_key: ENC[v1:...]       # opcional; recomendado em produção (ADR-004)
    users:                        # únicos usuários de leitura da v1 (ADR-003)
      - name: ana
        password_hash: "$2a$12$..."   # bcrypt custo >= 12; segredo no schema
        disabled: false
```
- O `n3x.yaml` do sidecar ganha o campo `controller.heartbeat_interval` (padrão de 10 s). Esse arquivo é contrato do HLD do `n3x-sidecar`.
- O core é implícito no sidecar e não se declara na seção `plugins`.

---

### Considerações de escalabilidade e disponibilidade
Abordagem geral
- Uma réplica do controller e escala vertical. O alvo é de 500 sidecars dentro de 150 MB e 0,15 vCPU para o core e os dados dele.
- O core nunca afeta o tráfego da aplicação: ele não tem hooks de requisição, e sem controller o sidecar entra em modo standalone.
- O estado se reconstrói pelos sidecars depois de um restart. A meta de volta ao painel em até 30 s vale nos dois casos até cerca de 200 serviços. A `session.hmac_key` reduz a volta para cerca de 10 s e evita a rajada de logins (ADR-004).

Técnicas aplicadas
- **Deduplicação:** as configs são guardadas pelo hash e os manifestos pelo hash do manifesto. Instâncias iguais não custam memória extra.
- **Teto de memória (ADR-009):**
  - Com uso acima de 90% do `memory_limit`, o core descarta nesta ordem: os sidecars desconectados mais antigos, depois os eventos mais antigos, depois as versões de config sem instância apontando.
  - Se mesmo assim o teto estourar, o core se marca `degradado` e o painel mostra o banner.
  - Degradado, ele continua aceitando heartbeats e mudanças de estado, mas recusa novas versões de config com `RESOURCE_EXHAUSTED`. O sidecar tenta de novo depois.
- **Snapshots imutáveis por serviço:** a escrita troca o ponteiro do snapshot, e o painel lê sem lock.
- **Controle de vazão:** fica com o host (1.000 conexões, 16 streams por conexão, mensagens de até 4 MiB e limites de login).
- **Rajadas de reconexão:** são suavizadas pelo jitter do login e pelo backoff do sidecar. Com a chave fixa, um restart nem gera novo login.
- **Heartbeat:** o processamento é O(1) por ping. O alvo é de 50 pings por segundo com 500 sidecars a cada 10 s.

Meta de disponibilidade
- Controller, e com ele a metade de controller do core: 99,5% ao mês (meta do sistema).
- Impacto do core no tráfego da aplicação: zero erros.
- Volta ao painel depois de um restart: cerca de 10 s com `hmac_key`. Sem ela, até 30 s enquanto houver até cerca de 200 serviços, limitado pelo bcrypt por serviço.

---

### Segurança
Autenticação
- **Sidecar:**
  - faz o login com o segredo do serviço (`ENC[...]`, decifrado pelo `.so`), conferido pelo host com bcrypt de custo 12 ou mais
  - recebe um token HMAC-SHA256 sobre serviço, instância, emitido em, expira em e nonce, com TTL de 1 h e renovação em 80% (ADR-005)
  - a validação do token é feita com `hmac.Equal`, em tempo constante
  - o token só existe na memória isolada do core e nenhum outro plugin o enxerga
- **Chave HMAC (ADR-004):**
  - pertence ao core
  - por padrão, 32 bytes de `crypto/rand` gerados no `Start`; opcionalmente, `session.hmac_key: ENC[...]`
  - sem a chave configurada, o `New` emite um aviso no log (`hmac_key_missing`)
  - a chave só é aceita em `ENC[...]`; em claro, a `Validate` recusa e o controller não sobe
- **Painel:**
  - o root é conferido pelo host
  - os usuários de leitura são conferidos pelo `CheckUser` do core contra os hashes do yaml
  - o bloqueio de 5 tentativas por 15 min fica no host

Autorização
- **Identidade amarrada ao token (ADR-005):** `Register`, `Heartbeat`, `Event` e `Exit` só aceitam a identidade igual à do token. Um sidecar do serviço A não consegue se apresentar como B. Se tentar, recebe `PERMISSION_DENIED`, e o controller gera um log de alerta.
- **Papéis:** só o papel `sidecar` chama o serviço `Core`, regra aplicada pelo host antes do handler.
- **Telas:** sidecars, configs e fluxo exigem o papel de leitura; usuários exige root.
- **Capacidades privilegiadas:** só são aceitas do ID `n3x.core` vindo de `github.com/giustech/n3x-plugin-core`, regra do SDK.

Proteção de dados
- Em trânsito: TLS 1.3 na conexão compartilhada e HTTPS no painel.
- Em repouso: não existe, porque tudo fica em memória. A `hmac_key` fica em `ENC[...]` (AES-256-GCM) e os `password_hash` são marcados como segredo no schema. Os dois ficam fora do painel e dos logs.
- Segredos de plugin nunca saem do sidecar: só sai a impressão digital do cifrado, ou `***:plain` com alerta.
- **Logs e erros:** nenhum token, segredo, hash de senha ou trecho de config sem máscara entra em log, erro gRPC ou evento. Isso é garantido por testes de varredura de segredos e pela análise estática do SDK.
- **LGPD:**
  - IP e nome de usuário só existem em memória, com até 24 h depois da desconexão
  - o acesso é só para quem está autenticado
  - não há trilha formal de auditoria na v1 (ADR-002); login, logout e bloqueio do painel ficam no log operacional do host

Gestão de segredos
- Os valores `ENC[...]` da seção do core são decifrados pelo host antes do `New` e chegam como `plugin.Secret`. O core nunca importa `secret/`.
- Do lado do sidecar, o `.so` decifra o `controller.secret` sozinho, como define o HLD do sidecar.
- **Rotação:**
  - `hmac_key`: trocar no yaml e reiniciar o controller, o que força um único novo login de todos
  - senha de usuário de leitura: novo hash no yaml e restart
  - segredo do serviço: segue o HLD de plataforma

---

### Observabilidade
Logs
- JSON por `log/slog`, com `plugin="n3x.core"`, serviço, instância e `x-request-id` quando houver.
- **Sidecar:**
  - resultado do login
  - registro enviado, com o hash
  - queda e reabertura do stream
  - recusas recebidas, com o código
  - `secret_in_clear`
- **Controller:**
  - registro aceito ou recusado, com o motivo
  - mudança de hash, com o hash anterior e o novo
  - conflito de identidade
  - divergência de config ou de manifesto
  - transição de estado
  - entrada e saída do modo degradado
  - aviso de `hmac_key` ausente
  - `PERMISSION_DENIED` por identidade trocada
- O nível padrão é `info`, e o heartbeat aparece só em `debug`.

Métricas
- **Controller (ADR-010, prefixo da ADR-022 do SDK):**
  - `n3x_n3x_core_sidecars{state}`
  - `n3x_n3x_core_divergences{service,kind=config|manifest}`
  - `n3x_n3x_core_identity_conflicts`
  - `n3x_n3x_core_registers_total{result}`
  - `n3x_n3x_core_logins_total{result}`
  - `n3x_n3x_core_events_total{kind}`
  - `n3x_n3x_core_memory_bytes`
  - `n3x_n3x_core_degraded`
  - `n3x_n3x_core_heartbeat_lag_seconds` (histograma)
- **Sidecar:** as métricas já definidas no HLD do sidecar, alimentadas pelo core: `n3x_controller_connected`, `n3x_controller_session_age_seconds`, `n3x_controller_logins_total{result}`, `n3x_controller_buffer_items` e `n3x_controller_buffer_dropped_total`.
- Latência e erros por RPC vêm do interceptor de métricas do host.
- O catálogo completo das famílias `n3x_n3x_core_*` (35 da metade de controller e 12 da de proxy) é o `internal/metricdefs` da FDD CORE-09; a lista acima é o núcleo. Entre as novas estão `token_rejections_total`, `hmac_key_configured`, `session_index_entries`, `session_invalidations_total`, `events_rejected_total`, `heartbeat_streams`, `config_versions` e `panel_build_seconds`.

Tracing
- Não há tracing distribuído na v1, porque são chamadas de controle fora do caminho da requisição.
- A correlação é feita pelo `x-request-id` e pelo par serviço e instância nos logs dos dois lados.

Dashboards e alertas
- **Cartões da visão geral:** sidecars por estado, divergências e alertas (conflitos, segredo em claro, degradado).
- **Alertas sugeridos ao operador:**
  - `n3x_n3x_core_degraded == 1` por 5 min
  - sidecars desconectados acima de 10% do total por 5 min
  - `n3x_n3x_core_identity_conflicts > 0`
  - divergência de config num serviço por mais de 30 min
  - taxa alta de `n3x_n3x_core_logins_total{result="fail"}`
- **Indicadores de SLO:**
  - percentual de sidecars conectados
  - tempo de reconstrução do inventário depois de um restart, com meta de 30 s ou menos

---

### Riscos arquiteturais e mitigação
#### Perda do inventário e das configs a cada restart do controller
- **Probabilidade:** alta
- **Impacto:** o painel fica vazio até os sidecars se reapresentarem, e o histórico de divergências e de hashes se perde
- **Mitigação:**
  - `session.hmac_key` fixa em produção, para não precisar de novo login
  - `resend_register` no ack do heartbeat
  - meta de 30 s medida na CI
- **Plano de contingência:** implementar um adaptador de banco na porta `Store` (v2, junto com a auditoria)

#### Ausência de auditoria formal na v1
- **Probabilidade:** média
- **Impacto:** não há trilha de quem acessou o painel, o que é uma lacuna de rastreabilidade e de LGPD
- **Mitigação:**
  - log operacional do host com login, logout e bloqueio
  - usuários só pelo yaml, então qualquer mudança passa pelo versionamento do repositório de deploy
- **Plano de contingência:** antecipar o FR-010 na v2 com banco, atrás da mesma porta `Store`

#### Estouro de memória com configs grandes ou muitos eventos
- **Probabilidade:** baixa
- **Impacto:** core degradado e novas versões de config recusadas
- **Mitigação:**
  - `memory_limit` de 128 MiB com contabilidade
  - deduplicação por hash
  - ordem de descarte definida
  - `Register` limitado a 1 MiB
  - métrica e alerta
- **Plano de contingência:** aumentar o `memory_limit`, ou reduzir as retenções e o limite de eventos

#### Mudança incompatível no proto quebrando sidecars antigos
- **Probabilidade:** média
- **Impacto:** sidecars sem registro e fora do painel, sem afetar o tráfego
- **Mitigação:**
  - `buf breaking` em todo PR
  - evolução só por adição
  - `proto_version` no `Register`
  - teste cruzado entre sidecar N-1 e controller N
- **Plano de contingência:** o controller aceita a major anterior durante uma janela de transição, a ser definida

#### Mascaramento deixando passar um segredo
- **Probabilidade:** baixa
- **Impacto:** alto, porque um segredo apareceria no painel ou num log
- **Mitigação:**
  - mascaramento dirigido pelo schema (`plugin.Secret`)
  - impressão digital calculada sobre o cifrado
  - `***:plain` com alerta
  - varredura de segredos na CI
  - teste de propriedade com campos de segredo aleatórios
- **Plano de contingência:** corrigir o mascaramento no sidecar, e limpar as configs em memória do controller com um restart (hipótese: ação de limpeza no painel na v2)

#### Conflito de identidade falso (pod reiniciando antes de o `Exit` chegar)
- **Probabilidade:** média
- **Impacto:** alerta falso no painel
- **Mitigação:**
  - o `Exit` marca o sidecar como desconectado na hora
  - o conflito só é declarado se as duas sessões mandarem heartbeat dentro do mesmo intervalo
- **Plano de contingência:** o conflito se resolve sozinho depois de 3 intervalos sem heartbeat da sessão antiga

---

### ADRs e próximos passos
ADRs associados
- [ADR-001 – Armazenamento só em memória na v1, atrás da porta `Store`](adr/ADR-001-armazenamento-em-memoria.md)
- [ADR-002 – Auditoria adiada para a v2](adr/ADR-002-auditoria-na-v2.md)
- [ADR-003 – Usuários de leitura só pelo `n3x.yaml`](adr/ADR-003-usuarios-pelo-yaml.md)
- [ADR-004 – Chave HMAC do core, aleatória ou `session.hmac_key`](adr/ADR-004-chave-hmac-do-core.md)
- [ADR-005 – Token de sessão com nonce, TTL de 1 h e identidade amarrada](adr/ADR-005-token-de-sessao.md)
- [ADR-006 – Config efetiva em JSON canônico, impressão digital de segredos e hash SHA-256](adr/ADR-006-config-efetiva-e-hash.md)
- [ADR-007 – Proto `Core` com heartbeat bidirecional e `proto_version`](adr/ADR-007-proto-core.md)
- [ADR-008 – Core sem hooks de requisição](adr/ADR-008-sem-hooks-de-requisicao.md)
- [ADR-009 – Teto de memória, ordem de descarte e estado degradado](adr/ADR-009-teto-de-memoria.md)
- [ADR-010 – Métricas `n3x_n3x_core_*` e sem tracing na v1](adr/ADR-010-metricas-e-tracing.md)
- Referenciados: ADR-003, ADR-005, ADR-016, ADR-017, ADR-019, ADR-020 e ADR-022 do `n3x-sdk`, e ADR-003 do `n3x-builder`

Decisões pendentes
- **Banco da v2 (Postgres ou outro):** decidir quando a auditoria voltar ou quando o controller precisar de mais de uma réplica.
- **Canal de drenagem no modo Istio:** já está pendente no HLD do sidecar. O core só consome o evento.
- **Janela de aceitação da major anterior do proto:** decidir na primeira quebra real de proto.

Próximos passos
- Escrever o `.proto` final e configurar `buf generate` e `buf breaking` no repositório
- Escrever o `n3x-plugin.yaml` do core: duas metades, `controller.required: true` e nenhuma fase
- Escrever as structs de `types/` das duas metades, com o schema gerado
- Implementar o JSON canônico, o mascaramento e o hash, com testes de propriedade e de varredura de segredos
- Implementar a porta `Store` e o adaptador `memory`, com contabilidade de bytes e o benchmark de 500 sidecars
- Montar as telas com widgets e testar acessibilidade (axe) no painel do controller
- Escrever o teste de ponta a ponta de restart do controller, com e sem `hmac_key`, medindo a meta de 30 s
- Escrever o LLD/FDD do core
