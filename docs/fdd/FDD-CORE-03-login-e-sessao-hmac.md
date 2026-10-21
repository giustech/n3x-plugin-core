### FDD: Login e sessão HMAC, as duas metades do core (CORE-03)

Versão: 1.0
Data: 2026-10-21
Responsável: Guilherme Giusti

---

### 1. Contexto e motivação técnica

**Problema técnico.** Sem sessão, nenhum plugin fala com o controller: o core é o único dono do interceptor da conexão compartilhada, do autenticador do servidor gRPC e da única rota pública, o `Login` (ADR-004 e ADR-019 do `n3x-sdk`). As peças em volta já estão fechadas: o verificador de credencial (FDD 31), o servidor gRPC e o mapeamento dos retornos do `Authenticate` (FDD 33), o proto `n3x.plugins.n3x.core.v1` com o serviço próprio `Login` (FDD 36), a conexão compartilhada com o teto do login e o buffer (FDD 38) e as capacidades privilegiadas `ConnInterceptors`, `Authenticate`, `PublicGRPC`, `SessionReporter` e `CallerIP` (FDD 09). Falta o comportamento que liga tudo:

- onde o token mora no sidecar (o HLD do core diz `env.Memory()`, que descarta itens em silêncio ao bater a cota, L-09)
- o formato exato do token no fio (campos, versão, codificação, tamanho) e como o `Authenticate` o confere em tempo constante, sem I/O e sem alocar no caminho comum
- como a chave HMAC nasce (aleatória no `Start` ou `session.hmac_key`), como é validada e como um token emitido por outra chave aparece no log
- quando o sidecar renova, como lida com relógios diferentes, o que o interceptor faz enquanto não há sessão e como uma recusa `UNAUTHENTICATED` vira um único novo login, sem rajada
- como a FDD 42 sabe que precisa reenviar o registro e como a FDD 43 obtém o "emitido em" da sessão, que o PRD pede no inventário (FR-006) mas a `ctrl.Identity` não carrega

**Encaixe no HLD.** Esta FDD especifica os componentes `agent/session` e `controller/auth` do HLD do `n3x-plugin-core` ("Componentes e responsabilidades"), a subida e o login de "Fluxo de requisição", o restart com e sem `session.hmac_key` de "Fluxos alternativos" e a parte de autenticação de "Segurança". Acrescenta um pacote interno `internal/sessiontoken`, só com biblioteca padrão, que as duas metades usam para o formato do token (hipótese de nome, H-01).

**Recorte com as FDDs vizinhas.**
- FDD 31: bcrypt, cache de credencial, deduplicação, fila e bloqueio por IP e por serviço. Aqui só a chamada a `Credentials().Verify` e o que fazer com cada retorno
- FDD 33: cadeia de interceptors do servidor, retirada do `authorization`, mapeamento para códigos e para `n3x_controller_auth_total`. Aqui só o `Authenticate` do core e os textos de motivo que vão para o log do host
- FDD 38: `grpc.ClientConn`, TLS, keepalive, teto de chamada, isenção do login, buffer e reconexão. Aqui o interceptor do core, o mais interno da cadeia, e o laço de login
- FDD 42: registro, heartbeat, eventos e saída no sidecar. Aqui só o gatilho (`Manager.Next`) que diz quando há sessão nova
- FDD 43: conferência de identidade em `Register`, `Heartbeat`, `Event` e `Exit`, inventário e conflito. Aqui só o `Authority.Session(nonce)` que entrega emitido em e expira em
- FDD 34: sessão do painel, sem relação com a sessão dos sidecars

**Atores.** Metade de proxy do core no `.so` (laço de login, guarda do token, interceptor), demais plugins com metade de controller (chamam pelo `env.Controller()` e recebem o token sem vê-lo), conexão compartilhada (FDD 38), servidor gRPC do controller e host de plugins (FDDs 32 e 33), verificador de credencial (FDD 31), metade de controller do core (handler `Login`, `Authenticate`, chave HMAC), operador (`session.ttl`, `session.hmac_key`, `controller.secret`), atacante na rede interna (token forjado, adulterado, reaproveitado ou de outra instância).

**Limites.**
- O controller não guarda tokens (ADR-003 do `n3x-sdk`, ADR-005 do core): só guarda, num índice limitado, o par (emitido em, expira em) de cada nonce já visto, para o inventário (contrato 4)
- O core não lê o `n3x.yaml`, não abre conexão e não importa `secret/`; recebe a chave como `plugin.Secret` aberta pelo host e a conexão pelo `env.Controller()`
- Nenhum outro plugin enxerga o token: ele fica num campo privado da instância do core e só sai pelo interceptor, num `ctx` derivado que só o transporte vê (FDDs 09 e 38)
- O tráfego da aplicação nunca depende da sessão: tudo roda em tarefa de fundo (ADR-008 do core, FR-002)

**Suposições**
- Go 1.26, sem cgo; `crypto/rand` não devolve erro no Go 1.26 (FDD 03)
- A conexão compartilhada aplica `min(prazo do ctx, controller.timeouts.login)` ao método `/n3x.plugins.n3x.core.v1.Login/Login` e `controller.timeouts.call` às demais unárias, antes do interceptor do core (FDD 38, L-08 fechada)
- O host chama `Authenticate` só para métodos fora da tabela pública e com o core fora de `failed`, e retira `authorization` dos metadados de entrada antes de qualquer handler, inclusive do `Login` (FDDs 09, 32 e 33)
- O controller é uma réplica única (ADR-001 do `n3x-sdk`); os tokens só precisam ser aceitos pelo processo que tem a mesma chave
- O relógio de parede do controller é sincronizado por NTP; o do sidecar pode estar adiantado ou atrasado sem afetar a sessão (contrato 2)

**Restrições**
- Token HMAC-SHA256 sobre serviço, instância, emitido em, expira em e nonce; TTL de 1 h configurável em `session.ttl`; renovação em 80% com jitter de 10% (ADR-005 do core)
- Validação em tempo constante com `hmac.Equal`; sem armazenamento de tokens (ADR-005 do core, ADR-003 do `n3x-sdk`)
- Chave do core aleatória no `Start` ou `session.hmac_key: ENC[...]`; rotação só com restart; `keys.hmac_file` não existe (ADR-004 do core)
- `Login` é a única rota sem sessão (FR-005, FR-011)
- `Authenticate` sem I/O, respeitando o `ctx`, sem o token no texto do erro e com 0 alocações no caminho comum (FDDs 09, 12 e 33)
- Nenhum token, segredo, chave ou valor de `authorization` em log, erro, evento ou métrica, nos dois processos

**Rastreabilidade**
- PRD 1.5: FR-002 (login com a credencial do serviço, token só no core, interceptor, renovação em 80% com jitter de 10%, novo login transparente, chamadas sem sessão esperam ou recebem `UNAVAILABLE`), FR-005 (HMAC-SHA256 sobre serviço, instância, emissão, expiração e nonce; TTL de 1 h; `session.hmac_key` ou 32 bytes aleatórios; `UNAUTHENTICATED` e `RESOURCE_EXHAUSTED`; restart com e sem chave; rotação), FR-006 (horário de emissão da sessão no inventário e idade da sessão no detalhe), FR-011 (login como única rota pública), FR-016 (identidade por instância); não funcionais de disponibilidade (volta ao painel em até 30 s) e segurança (nenhum token em log); critérios de aceite "credencial inválida recusada com `UNAUTHENTICATED`, assim como chamada sem sessão válida" e "controller desligado e religado com o sidecar de volta ao painel em até 30 s"
- HLD do `n3x-plugin-core`: `agent/session` e `controller/auth`, "Fluxo de requisição" (subida e login), "Fluxos alternativos" (restart com e sem `hmac_key`), "Segurança", "Observabilidade"
- ADRs: ADR-004 e ADR-005 do `n3x-plugin-core`; ADR-003, ADR-004, ADR-016 e ADR-019 do `n3x-sdk`; ADR-004 do `n3x-controller`; ADR-006 do `n3x-sidecar`
- FDDs aplicadas: 03 (`plugin.Secret`, `NewSecret`, `InClear`, `Sealed`, formato do `keygen`), 04 (sem modo fixo no `memory.Map`), 06 (`proxy.Sleep`, `Background`, `telemetry`, `x-request-id`), 08 (`Identity`, `Validator`, `Session` igual ao nonce), 09 (contratos 3, 4, 5, 9, 10, 11, 14 e 17), 12 (`ctrltest`, `Link`, `FakeCredentials`, `hostconformance.RunCoreEnv` com `WithCore`, `sdktest.AssertZeroAllocsFunc`), 30 (seção `plugins["n3x.core"]`, `keygen`, `encrypt --plugin`), 31 (retornos do `Verify`, cache por serviço, P-03), 32 (`Host.Authenticate`, `ErrMisbehavior`), 33 (mapeamento de `Authenticate`, `auth_failed.detail`, cabeçalhos até 16 KiB), 36 (proto, `corev1.MaxTokenLen`, `types.SessionConfig`, regra `hmac_key_format`, logs `hmac_key_missing` e `hmac_key_in_clear`, versão mínima substituída), 38 (`LoginMethod`, teto, interceptor sem teto, buffer, reconexão, P-01), 40 (composição da `Validate`), 41 (`SessionRecord` com `IssuedAt` e `ExpiresAt`)
- Lacunas do README:
  - **L-09 fechada aqui:** o token fica num campo privado da instância do core (`atomic.Pointer` no `session.Manager`), nunca em `env.Memory()` (contrato 6; atende o pedido da FDD 04)
  - **L-08 aplicada** (fechada na FDD 38): o teto é da visão do `Env`; o interceptor do core não impõe teto; o login escapa do teto de 5 s pelo método completo e usa `controller.timeouts.login` (10 s)
  - **L-02 aplicada** (fechada na FDD 09, implementada na FDD 33): o IP do `Verify` vem de `CallerIP(ctx)`; sem IP, `netip.Addr{}`, que o verificador põe no balde `unknown` (FDD 31)
  - **L-11, L-12, L-13, L-34 e L-64 aplicadas:** serviço `Login` próprio; métricas de sessão do sidecar pelo `SessionReporter`; `x-request-id` no login; caminhos `plugins["n3x.core"].session.hmac_key`; `hmac_key` em claro aceita com aviso
- Pedidos de FDDs anteriores atendidos: FDD 03 (aviso de `hmac_key` em claro; chave como base64 de 32 bytes do `keygen`), FDD 04 (token fora do `memory.Map`), FDD 08 (regras da sessão na `Validate`, aviso no `New`, `Session` igual ao nonce), FDD 09 (0 alocações no `Authenticate`, chamadas ao `SessionReporter`, classificação dos códigos, P-05), FDD 12 (`AssertZeroAllocsFunc` sobre `Authenticate`, testes com `ctrltest` e `Link`), FDD 16 (dados de sessão pela capacidade), FDD 31 (`NewSecret` sem log, sentinelas devolvidas como estão, jitter de 1 s a 60 s com primeira espera curta, renovação que acerta o cache), FDD 33 (`Authenticate` sem I/O, respeitando o `ctx`, `ErrUnauthenticated` sem token, token pequeno), FDD 36 (`hmac_key_format`, avisos no `New`, interceptor que pula o `Login`, P-02 e P-05), FDD 38 (interceptor sem teto, comportamento sem sessão, P-01), FDD 40 (sessão antes dos usuários na `Validate`)

---

### 2. Objetivos técnicos

- **Token invisível fora do core.** Invariante: o token só existe no `session.Manager` do sidecar, no `ctx` derivado que o interceptor entrega ao transporte e no fio; nenhum `memory.Map`, log, erro, evento ou métrica o contém. Medida: com servidor que ecoa metadados, o `ctx` do plugin não tem `authorization` depois da chamada, e uma varredura pelo texto exato do token devolvido no login acha 0 ocorrência em todos os logs e erros capturados dos dois lados.
- **`Authenticate` barato e seguro.** Medida: 0 alocações por chamada com token válido (`sdktest.AssertZeroAllocsFunc`, passo sem `-race`); p99 de no máximo 20 µs no benchmark com 8 goroutines (meta da FDD 09); comparação do MAC só por `hmac.Equal`; nenhuma chamada de I/O (conferido por teste com `ctx` cancelado e por revisão).
- **Formato pequeno e determinístico.** Medida: token de no máximo 422 bytes (418 com datas de 10 dígitos, até o ano 2286) e valor de `authorization` de no máximo 429 bytes (limite de parse de 1.024, `corev1.MaxTokenLen`); mesma chave e mesmas entradas dão os mesmos bytes, conferido por vetores golden; fuzzing de 5 minutos do parser sem panic na CI noturna.
- **Renovação no ponto certo, com qualquer relógio no sidecar.** Medida: com TTL de 1 h e relógio falso, a renovação acontece entre 43,2 e 52,8 min depois do login, e exatamente em 48 min com jitter zero; com o relógio do sidecar 30 min adiantado ou atrasado, o ponto não muda (a duração vem do próprio token).
- **Um único novo login por recusa.** Invariante: N chamadas recusadas com `UNAUTHENTICATED` pela mesma geração de token disparam exatamente 1 `Login`, que começa entre 0 e 1 s depois da primeira recusa.
- **Restart do controller sem rajada desnecessária.** Medida: com `session.hmac_key` fixa, um restart não gera nenhum `Login` (contador `n3x_controller_logins_total` igual antes e depois); sem a chave, cada sidecar faz exatamente 1 `Login` depois da primeira recusa, e o `Authenticate` do controller novo registra o motivo `key` (token de outra chave).
- **Credencial longa só no login.** Invariante: `controller.secret` só sai do sidecar no corpo do `Login`; o controller o embrulha em `plugin.NewSecret` e o entrega ao `Verify` sem log, sem métrica e sem guardar.
- **Falha fechada.** Invariante: sem chave carregada, token malformado, de outra versão, de outra chave, adulterado, expirado ou emitido no futuro além da tolerância, o `Authenticate` recusa; nunca há caminho que aceite sem MAC conferido.

---

### 3. Escopo e exclusões

**Incluído**
- Pacote `internal/sessiontoken` (só biblioteca padrão): formato `v1` do token, cabeçalho `Bearer`, `Key` com `Sign` e `Verify`, `Inspect` sem chave, vetores golden e fuzzing (contrato 2)
- `controller/auth`: `Authority` com a chave (configurada ou aleatória), handler `Login` da tabela pública, emissão do token, `Authenticate`, índice limitado de sessões vistas para a FDD 43, `ValidateSession` com a regra `hmac_key_format`, avisos `hmac_key_missing` e `hmac_key_in_clear`, métricas e logs (contratos 1, 3, 4 e 5)
- Ligação na metade de controller: `controller.New`, `Start`, `PublicGRPC`, `Authenticate` e a composição `Config.Validate` (sessão antes dos usuários) (contrato 5)
- `agent/session`: `Manager` com o laço de login (tarefa de fundo `session`), guarda do token em campo privado, interceptors unário e de stream entregues por `ConnInterceptors()`, renovação em 80% com jitter de 10%, novo login com backoff e jitter, detecção de recusa, `SessionReporter`, e o gatilho `Next` para a FDD 42 (contratos 6, 7 e 8)
- Comportamento das chamadas dos plugins enquanto o login não completa e durante a troca de token
- Seção `plugins["n3x.core"].session` (`ttl`, `hmac_key`) do `n3x.yaml` do controller: semântica, validação e avisos
- Testes: unidade com relógio falso, kit de proxy (`sdktest`) e de controller (`ctrltest`, `Link`, `hostconformance.RunCoreEnv` com `WithCore`), vetores, fuzzing, canário de segredos e benchmark

**Excluído**
- Bcrypt, cache de credencial, deduplicação, fila e bloqueio por IP e por serviço: FDD 31 (CTL-02)
- Cadeia de interceptors do servidor, retirada do `authorization`, mapeamento para códigos e `n3x_controller_auth_total`, IP do chamador: FDD 33 (CTL-04)
- `grpc.ClientConn`, TLS, keepalive, teto de chamada, buffer e reconexão: FDD 38 (SID-10)
- Registro, heartbeat, eventos e saída no sidecar, inclusive o que fazer depois de cada sessão nova: FDD 42 (CORE-06)
- Conferência de identidade em `Register`, `Heartbeat`, `Event` e `Exit`, conflito de identidade e inventário: FDD 43 (CORE-07)
- Sessão do painel, root e login do painel: FDD 34 (CTL-05)
- Cartões e consolidação das métricas `n3x_n3x_core_*`: FDD 45 (CORE-09)
- Fora da v1: rotação da chave sem restart (aceitar a chave anterior por um prazo), revogação de uma sessão específica, várias réplicas do controller, token amarrado ao IP ou à conexão, mTLS de sidecar

---

### 4. Fluxos detalhados e diagramas

**Fluxo de subida do controller (metade de controller do core)**
1. Fase de config (FDD 08, contrato 15 da FDD 09): `Config.Validate()` roda `auth.ValidateSession(c.Session)` e depois `directory.ValidateUsers(c.Users)` (ordem dos campos da struct, FDD 40). A regra `hmac_key_format` confere a chave aberta; com `--skip key`, o texto ainda é um envelope `ENC[...]` e a regra é pulada.
2. `controller.New`: `auth.New(auth.Options{Session: cfg.Session, Metrics, Log})`. Com `hmac_key` presente, decodifica os 32 bytes e guarda a origem `config`; se o texto veio sem `ENC[...]` (`InClear()`), loga `hmac_key_in_clear` (warn). Sem `hmac_key` (ausente ou vazia, `IsZero()`), marca a origem `random` e loga `hmac_key_missing` (warn). Cria as métricas (`New` é o único lugar em que o SDK aceita criar famílias).
3. `controller.Start`: `a.Start()`. Com origem `random`, lê 32 bytes de `crypto/rand`. Deriva o identificador da chave (`kid`, 8 hex), monta a lista livre de estados de HMAC e publica a autoridade num ponteiro atômico. Loga `session_authority_ready` com `key_source`, `kid` e `ttl`.
4. `PublicGRPC`: registra `corev1.RegisterLoginServer(r, a.LoginServer(cenv))` na tabela pública (FDD 09, contrato 10).
5. O host liga `core.Authenticate` ao interceptor de autenticação (FDD 33) depois que todos os plugins sobem.

**Fluxo principal do login no controller** (`/n3x.plugins.n3x.core.v1.Login/Login`)
1. O host reconhece o método da tabela pública: não chama `Authenticate`, retira `authorization`, resolve o IP, põe o `request_id` e chama o handler com `Caller` zero e prazo `min(cliente, plugin_call_timeout)` (FDD 33).
2. `ctx.Err()` já preenchido: devolve o erro de `context` como está (a FDD 33 responde `DEADLINE_EXCEEDED` ou `CANCELED`).
3. Confere os limites do contrato 1 da FDD 36: `service` na gramática da FDD 15 (1 a 63 bytes), `instance` com 1 a 253 bytes ASCII visível (0x21 a 0x7E), `secret` com 1 a 1.024 bytes. Fora disso, `INVALID_ARGUMENT` com o nome do campo e sem o valor; o `Verify` não é chamado e nada conta como falha de credencial (hipótese, H-13).
4. `ip, ok := cenv.CallerIP(ctx)`; sem IP, `netip.Addr{}` (balde `unknown` da FDD 31).
5. `err := cenv.Credentials().Verify(ctx, service, plugin.NewSecret(secret), ip)`. Erro: devolve como veio, sem embrulho (`ErrInvalidCredentials`, `ErrLoginBlocked`, `ErrVerifierBusy` têm `GRPCStatus()`; `ctx.Err()` é mapeado pelo host). Nenhum log do core por tentativa: o verificador já loga `login_failed` e `login_blocked` amostrados (FDD 31).
6. Emissão: `iat` = segundo atual do relógio de parede (Unix, truncado); `exp = iat + floor(session.ttl em segundos)`; nonce de 16 bytes de `crypto/rand`; `token = key.Sign(...)` no formato do contrato 2.
7. Grava (nonce, `iat`, `exp`) no índice de sessões (contrato 4) e devolve `LoginResponse{token, expires_at = exp}`. Conta `logins_total{result="ok"}` e loga `session_issued` em `debug`, sem nonce nem token.

**Fluxo do `Authenticate`** (toda chamada fora da tabela pública, FDD 33)
1. `ctx.Err() != nil`: devolve `ctx.Err()` (motivo `timeout` no host).
2. Autoridade ainda não publicada: recusa com o motivo `not_ready` (não acontece depois do `Start`; defesa).
3. Tira o prefixo `Bearer ` (esquema comparado sem diferenciar caixa); valor sem o prefixo, vazio ou com mais de 1.024 bytes depois dele: `malformed`.
4. Lê os campos por posição, sem `strings.Split`: versão, `kid`, `iat`, `exp`, nonce e serviço da esquerda para a direita (nenhum deles contém `.`), MAC nos 43 últimos caracteres e instância no meio. Forma errada: `malformed`; versão diferente de `v1`: `version`.
5. `kid` diferente do da chave atual: `key` (token de outra chave, típico de restart sem `hmac_key` ou de chave trocada). O `kid` não é segredo e não escolhe chave na v1.
6. Copia o texto assinado para o buffer do estado de HMAC tirado da lista livre, calcula `HMAC-SHA256(chave, rótulo || texto)`, decodifica o MAC recebido (base64url estrito) e compara com `hmac.Equal`. Diferente: `signature`.
7. Só depois do MAC: `exp - iat` fora de 5 min a 24 h: `lifetime`; `iat > agora + 2 min`: `not_yet_valid`; `agora >= exp`: `expired`.
8. Devolve `Identity{Role: RoleSidecar, Service, Instance, Session: nonce}`, com as três strings apontando para dentro do próprio texto do token (0 alocações). Nonce que não está no índice entra nele (só a primeira vez de cada sessão aloca).
9. O host confere a identidade, retira `authorization`, faz `ContextWithCaller` e segue (FDDs 32 e 33). Nenhum erro do core leva o token; o texto do motivo vai para o `detail` do `auth_failed` do host.

**Fluxo principal no sidecar: subida e primeiro login**
1. `agent.New` (instância `core`, FDD 09): com `env.Controller()` devolvendo `ErrStandalone`, não cria tarefas e os interceptors são de passagem. Com controller, cria o `session.Manager` com a conexão, a identidade (`Sidecar().Identity()`), o `ControllerSettings` (segredo e `LoginTimeout`) e o `Sidecar().Session()`, e registra `env.Background("session", m.Run)`.
2. O executor chama `ConnInterceptors()` uma vez e instala o par como o mais interno da cadeia (FDDs 17 e 38); o core devolve `m.Interceptors()`.
3. As tarefas de fundo começam depois da subida inteira (FDD 06). `m.Run` faz o primeiro login na hora: `ctx` da tarefa com prazo `LoginTimeout`, `grpc.WaitForReady(true)` e `corev1.NewLoginClient(conn).Login(ctx, {service, instance, secret.Reveal()})`. A visão aplica o teto de login (10 s); o interceptor do core reconhece o método e passa sem token.
4. Sucesso: confere a resposta (token de 1 a 1.024 bytes ASCII visível, `expires_at` presente), lê `iat` e `exp` do próprio token com `sessiontoken.Inspect` (sem chave), calcula a duração `L = exp - iat` e publica o estado novo no ponteiro atômico: geração 1, cabeçalho `Bearer <token>` pronto, `kind = first`. Chama `LoginAttempt(LoginOK)` e `SessionStarted(iat)`, loga `session_login_ok` e acorda quem espera em `Next`.
5. Agenda a renovação em `0,8 x L x (1 + u)` com `u` uniforme entre -0,1 e +0,1, descontado o tempo que o login levou; agenda a expiração local em `L` menos esse mesmo tempo. As esperas usam `proxy.Sleep`, que o kit troca pelo relógio falso.

**Fluxo da chamada de um plugin (interceptor unário)**
1. A visão do `Env` já aplicou marcas, namespace, teto, metadados e `x-request-id` (FDD 38).
2. Método igual a `corev1.Login_Login_FullMethodName`: chama o `invoker` sem token e sem esperar.
3. Estado com token válido (ativo ou em renovação): sobrescreve `authorization` com o cabeçalho da geração atual num `ctx` derivado e chama o `invoker`.
4. Sem token válido e com login em andamento: espera o fim da tentativa ou o fim do `ctx`. Login concluiu: segue como no passo 3. Login falhou: `UNAVAILABLE` "controller: sessão não estabelecida". Prazo do `ctx` venceu: `UNAVAILABLE` com o mesmo texto; `ctx` cancelado: `CANCELED`.
5. Sem token válido e sem login em andamento (espera de backoff, sessão recusada ainda não refeita, gerenciador parado): `UNAVAILABLE` na hora, sem rede. É o "recebem `UNAVAILABLE`" do FR-002; as chamadas com `proxy.Buffered()` ficam no buffer e repetem com espera própria (FDD 38).
6. Resposta com código `UNAUTHENTICATED` e a mensagem genérica do host ("sessão ausente, inválida ou expirada"): marca a geração usada como recusada (uma vez por geração) e acorda o laço de login. Outros códigos passam sem efeito.

**Fluxo do stream** (heartbeat da FDD 42 ou stream de outro plugin)
- Mesmo tratamento dos passos 2 a 5 na abertura. O stream é embrulhado para observar o erro de `Header()` e de `RecvMsg`: `UNAUTHENTICATED` com a mensagem genérica marca a geração usada na abertura como recusada.
- O token vale para a abertura do stream (o servidor autentica uma vez, FDD 33). Depois de uma renovação, quem abriu o stream decide reabri-lo com o token novo (pedido à FDD 42); o servidor encerra streams cuja sessão expirou (pedido à FDD 43).

**Fluxo da renovação**
1. No ponto de renovação, o laço faz um `Login` completo com a mesma credencial (o proto só tem `Login`; não há RPC de renovação). No controller, o `Verify` acerta o cache por serviço enquanto `verified_cache_ttl` não venceu (FDD 31): com `session.ttl` de 1 h e cache de 1 h, a primeira renovação de cada pod costuma acertar e o custo em regime é de no máximo um bcrypt por serviço por `verified_cache_ttl`.
2. Durante a renovação o token antigo continua em uso: chamadas em voo e novas chamadas levam o token antigo, válido por pelo menos 12% do TTL (7,2 min com 1 h). O estado novo entra por troca atômica do ponteiro; as chamadas iniciadas depois levam o token novo. Nenhuma chamada espera a renovação e nenhuma falha por causa dela.
3. Sucesso: geração nova com `kind = renewed`, `SessionStarted(iat novo)`, acorda `Next`.
4. Falha (código transitório, credencial recusada, bloqueio): o token antigo segue até a expiração local; o laço repete com backoff (1 s, multiplicador 1,6, teto 60 s, jitter de 20%), nunca depois da expiração local. Na expiração local sem sucesso: estado sem token, `SessionEnded()`, `session_invalidations_total{cause="expired"}`, log `session_expired`, e o laço segue tentando como no novo login.

**Fluxo do novo login depois de recusa** (`UNAUTHENTICATED` numa chamada que levou token)
1. A primeira recusa da geração `g` muda o estado para "recusado" por comparação e troca; as demais recusas da mesma geração não fazem nada. `SessionEnded()`, `session_invalidations_total{cause="rejected"}` e log `session_rejected` (no máximo 1 por minuto, com a contagem).
2. O laço espera um tempo uniforme entre 0 e 1 s (espalha a rajada do restart sem chave) e faz o `Login`, com `kind = relogin`.
3. Se o token emitido há menos de 30 s for recusado de novo 3 vezes seguidas (relógio do controller errado, chave trocando, bug), a espera passa a seguir o backoff completo até 60 s e sai o log `session_rejected_fresh` (warn), para não virar laço de login.
4. Depois do sucesso, a FDD 42 recebe a geração nova por `Next` e reenvia o `Register` (FR-002, "reenvia o registro completo").

**Fluxos alternativos e exceções**
- **Controller fora do ar na subida:** o `Login` espera a conexão (`WaitForReady`) até o teto de 10 s, falha com `DEADLINE_EXCEEDED` ou `UNAVAILABLE` (`LoginError`) e repete com backoff de 1 s a 60 s e jitter de 20%. O tráfego segue; os envios com `Buffered` esperam no buffer; as chamadas síncronas recebem `UNAVAILABLE` enquanto não há login em andamento.
- **Queda e reconexão da conexão, controller com a mesma chave:** o token continua válido; o core não faz login novo ao reconectar. As chamadas seguintes passam; o heartbeat reaberto pela FDD 42 recebe `resend_register` do controller novo (FDD 43). Nenhum `Login`.
- **Restart do controller sem `session.hmac_key` (ou com chave trocada):** a primeira chamada depois da volta (heartbeat reaberto, evento do buffer ou `Register`) recebe `UNAUTHENTICATED`; o `Authenticate` do controller novo registra o motivo `key`; o sidecar segue o fluxo de novo login (0 a 1 s de espera, um `Login`, `Register`). Com 50 serviços, o custo no controller é de cerca de um bcrypt por serviço, pela deduplicação e pelo cache da FDD 31.
- **Credencial errada no sidecar:** `UNAUTHENTICATED` "credencial inválida" no `Login` (`LoginFail`); backoff até 60 s; o verificador bloqueia o IP depois de 5 falhas e passa a responder `RESOURCE_EXHAUSTED` sem bcrypt (FDD 31). Log `session_login_failed` com `reason=credential` (primeira falha e depois no máximo 1 por minuto).
- **`RESOURCE_EXHAUSTED` no login (bloqueio ou verificador ocupado):** `LoginFail`; primeira espera de cerca de 1 s (o ocupado é passageiro), depois backoff até 60 s (P-03 da FDD 31).
- **`FAILED_PRECONDITION` (controller de outra major) ou `UNIMPLEMENTED` (core mínimo `v0.1.0` no controller):** `LoginError`; espera direto de 60 s com jitter; log `session_login_failed` com o código.
- **Resposta de login inválida** (token vazio, acima de 1.024 bytes, com caractere fora de 0x21 a 0x7E, sem `expires_at`): `LoginError`, log `session_response_invalid` (error), backoff.
- **Token que o `Inspect` não entende** (controller com um core mais novo e outro formato): usa `L = expires_at - agora local`, limitado a 1 min a 24 h, e loga `session_token_opaque` (warn) uma vez por processo; o sidecar continua funcionando, só fica sensível à diferença de relógio.
- **Panic na tarefa `session`:** o runtime reinicia a tarefa (FDD 06, espera de 1 s a 60 s); o estado do `Manager` sobrevive; com sessão ativa, o laço reinicia com uma renovação imediata (o acerto de cache torna barato), porque perdeu a contagem do tempo.
- **Panic no interceptor:** a guarda da FDD 38 devolve `INTERNAL` e conta em `n3x_plugin_panics_total{plugin="n3x.core",instance="core"}`.
- **Desligamento:** o `ctx` da tarefa `session` é cancelado (FDD 06): o laço para, não renova mais e chama `SessionEnded()`, mas o token continua no estado para o `Exit` da FDD 42 usar dentro do `ExitContext()` de 1,5 s. Se o `Exit` precisar de login (sem token), espera um login em andamento só até o prazo do `ExitContext`.
- **Standalone:** nenhuma tarefa; `ConnInterceptors()` não é chamado; `SessionReporter` não é chamado; as séries de sessão ficam em zero (FDD 09).
- **Controller: chave configurada inválida no `New`** (só sem a `Validate`, em teste, ou envelope dentro de envelope): `auth.ErrInvalidKey`; o `New` do core falha e o controller não sobe (saída 5, core obrigatório).
- **Controller: relógio de parede voltou mais de 2 min:** tokens recentes ficam com `iat` no futuro e são recusados com `not_yet_valid`; os sidecars refazem o login uma vez com o relógio novo.

**Diagramas**

Login, chamada autenticada e renovação:

```mermaid
sequenceDiagram
    participant S as session.Manager (sidecar)
    participant I as interceptor do core
    participant G as conexão (FDD 38)
    participant H as host (FDD 33)
    participant A as controller/auth
    participant V as Verify (FDD 31)
    S->>I: Login(serviço, instância, segredo) com WaitForReady
    I->>G: método de login: sem token
    G->>H: teto de 10 s, x-request-id
    H->>A: handler Login, Caller zero
    A->>V: Verify(ctx, serviço, NewSecret(segredo), CallerIP)
    V-->>A: nil
    A->>A: iat, exp, nonce, HMAC-SHA256, índice
    A-->>S: token e expires_at
    S->>S: Inspect: L = exp - iat; estado geração 1; Next acorda a FDD 42
    Note over S: plugin chama o próprio serviço
    I->>G: authorization: Bearer v1...
    H->>A: Authenticate(ctx, valor)
    A-->>H: Identity{sidecar, serviço, instância, nonce}
    Note over S: 0,8 x L x (1 +- 0,1)
    S->>I: Login de renovação (token antigo segue em uso)
    A->>V: Verify: acerto de cache
    A-->>S: token novo; troca atômica; geração 2
```

Restart do controller sem `session.hmac_key`:

```mermaid
sequenceDiagram
    participant S as sidecar (core)
    participant K as controller novo
    S->>K: Heartbeat reaberto (FDD 42) com o token antigo
    K->>K: Authenticate: kid diferente, motivo key
    K-->>S: UNAUTHENTICATED "sessão ausente, inválida ou expirada"
    S->>S: geração recusada (uma vez); espera 0 a 1 s
    S->>K: Login
    K-->>S: token novo (um bcrypt por serviço, deduplicado)
    S->>K: Register (FDD 42, por Next)
```

Estados do `session.Manager`:

```mermaid
stateDiagram-v2
    state "sem sessão, login em andamento" as Logging
    state "sem sessão, esperando backoff" as Backoff
    state "ativo" as Active
    state "ativo, renovando" as Renewing
    state "parado" as Stopped
    [*] --> Logging: Run começa
    Logging --> Active: Login OK
    Logging --> Backoff: Login falhou
    Backoff --> Logging: espera vencida
    Active --> Renewing: 80% do TTL com jitter
    Renewing --> Active: Login OK (troca atômica)
    Renewing --> Renewing: falha, repete antes da expiração local
    Renewing --> Backoff: expiração local sem sucesso
    Active --> Backoff: UNAUTHENTICATED (espera 0 a 1 s)
    Active --> Backoff: expiração local
    Logging --> Stopped: ctx cancelado
    Backoff --> Stopped: ctx cancelado
    Active --> Stopped: ctx cancelado (token mantido para o Exit)
    Renewing --> Stopped: ctx cancelado
    Stopped --> [*]
```

Decisão do `Authenticate`:

```mermaid
flowchart TD
    A[ctx vencido?] -- sim --> X0[ctx.Err: timeout no host]
    A -- não --> B[autoridade publicada?]
    B -- não --> X1[not_ready]
    B -- sim --> C[prefixo Bearer, 1 a 1024 bytes, campos por posição?]
    C -- não --> X2[malformed]
    C -- sim --> D[versão v1?]
    D -- não --> X3[version]
    D -- sim --> E[kid igual ao da chave?]
    E -- não --> X4[key]
    E -- sim --> F[hmac.Equal do MAC?]
    F -- não --> X5[signature]
    F -- sim --> G[exp - iat de 5 min a 24 h?]
    G -- não --> X6[lifetime]
    G -- sim --> H[iat até agora + 2 min?]
    H -- não --> X7[not_yet_valid]
    H -- sim --> I[agora antes de exp?]
    I -- não --> X8[expired]
    I -- sim --> OK[Identity sidecar com substrings do token]
```

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

Os contratos 1, 2 e 5 são públicos para quem fala com o controller e para o operador. Os contratos 3, 4, 6, 7 e 8 são API Go interna do módulo `github.com/giustech/n3x-plugin-core`, estáveis para as FDDs 42, 43 e 45, sem garantia de semver para fora; o 3 implementa a interface pública `ctrl.Authenticator` da FDD 09 e segue a semântica dela.

**Contrato 1: RPC `Login` (`n3x.plugins.n3x.core.v1.Login/Login`)**
- Tipo: endpoint gRPC (protobuf, tabela pública, porta `17443`)
- Assinatura/Rota: `/n3x.plugins.n3x.core.v1.Login/Login` (`corev1.Login_Login_FullMethodName`); mensagens do contrato 1 da FDD 36
  ```proto
  service Login { rpc Login(LoginRequest) returns (LoginResponse); }
  message LoginRequest  { string service = 1; string instance = 2; string secret = 3; }
  message LoginResponse { string token = 1; google.protobuf.Timestamp expires_at = 2; }
  ```
- Método: unário, sem `authorization`, com `x-request-id` (FDD 38)
- Semântica de status/headers:
  - **Campos:** `service` é o `service` do `n3x.yaml` do sidecar (gramática da FDD 15, 1 a 63 bytes); `instance` é a identidade da instância detectada pelo pré-voo (FDD 25), 1 a 253 bytes ASCII visível sem espaço (regra da FDD 36, mais larga que a gramática da FDD 25, então o controller nunca recusa uma instância detectada); `secret` é o `controller.secret` aberto pelo `.so`, 1 a 1.024 bytes, nunca logado
  - **`token`:** formato `v1` do contrato 2, de 1 a 1.024 bytes (`corev1.MaxTokenLen`), opaco para os outros plugins; o sidecar só lê `iat` e `exp` dele para agendar
  - **`expires_at`:** `exp` do token, UTC, com nanos zero; informativo para o sidecar (o agendamento usa `exp - iat`, contrato 8)
  - **Identidade:** a instância é a que o cliente declara; quem tem o segredo do serviço pode pedir sessão para qualquer instância daquele serviço (credencial por serviço, ADR-003 do `n3x-sdk`); um serviço nunca obtém sessão de outro
  - **Metadados de entrada:** o host retira `authorization` também aqui (FDD 33); o handler não vê token
  - **Prazo:** `min(prazo do cliente, runtime.plugin_call_timeout)` no servidor (10 s); 10 s no cliente (`controller.timeouts.login`); o handler não tem prazo próprio
  - **Custo:** sem contar o `Verify`, até 1 ms no p99 (16 bytes de `crypto/rand`, um HMAC, formatação); com o `Verify`, o custo é o da FDD 31 (cerca de 250 ms num bcrypt de custo 12, quase nada no acerto de cache)

| Situação | Código | Mensagem | `n3x_n3x_core_logins_total{result}` | Quem decide |
| --- | --- | --- | --- | --- |
| Credencial aceita | `OK` | | `ok` | core |
| Campo fora dos limites | `INVALID_ARGUMENT` | `login: campo <service\|instance\|secret> fora do formato` | `invalid_request` | core (sem `Verify`) |
| Segredo errado, serviço desconhecido | `UNAUTHENTICATED` | credencial inválida | `fail` | `Verify` (`ErrInvalidCredentials`) |
| IP ou serviço bloqueado | `RESOURCE_EXHAUSTED` | muitas tentativas; tente mais tarde | `blocked` | `Verify` (`ErrLoginBlocked`) |
| Sem vaga de bcrypt | `RESOURCE_EXHAUSTED` | controller ocupado; tente mais tarde | `busy` | `Verify` (`ErrVerifierBusy`) |
| Prazo ou cancelamento | `DEADLINE_EXCEEDED` ou `CANCELED` | texto do `context` | `timeout` | `Verify` ou `ctx` (mapeado pelo host) |
| Autoridade não publicada ou falha ao gerar o nonce | `UNAVAILABLE` | core: sessão indisponível | `error` | core (defesa) |
| Outra major do proto | `FAILED_PRECONDITION` | texto da FDD 33 | (não chega ao core) | host |
| Core `failed` | `UNAVAILABLE` | plugin indisponível | (não chega ao core) | host |

**Exemplo de requisição**
```text
:path /n3x.plugins.n3x.core.v1.Login/Login
x-request-id: 0f6a2d1e-9c3b-4e57-8a10-5b7c9d2e4f68
(mensagem) {"service":"pedidos","instance":"pedidos-7d9f8b-xk2lp","secret":"<segredo de 43 caracteres do credential new>"}
```

**Exemplo de resposta**
```json
{
  "token": "v1.3f9a0c1e.1792591201.1792594801.q3Jx9kLm2Vb8Tz1Rw5Yc0A.pedidos.pedidos-7d9f8b-xk2lp.kV2xQ7nP0aZ3cR8tW5yB1mL6dF9hJ4sG2uE7oI0rT3w",
  "expiresAt": "2026-10-21T15:00:01Z"
}
```

**Exemplo de resposta** (recusas)
```text
rpc error: code = Unauthenticated desc = credencial inválida
rpc error: code = ResourceExhausted desc = controller ocupado; tente mais tarde
rpc error: code = InvalidArgument desc = login: campo instance fora do formato
```

---

**Contrato 2: formato `v1` do token e metadado `authorization` (`internal/sessiontoken`)**
- Tipo: sdk (formato de fio e API Go interna do core, só biblioteca padrão; usada pelas duas metades)
- Assinatura/Rota:
  ```text
  token  = "v1" "." kid "." iat "." exp "." nonce "." service "." instance "." mac
  kid    = 8 hex minúsculos: 4 primeiros bytes de HMAC-SHA256(chave, "n3x.core/session/kid" 0x00)
  iat    = segundos Unix da emissão, decimal sem zero à esquerda (relógio do controller)
  exp    = segundos Unix da expiração, decimal sem zero à esquerda
  nonce  = 16 bytes de crypto/rand em base64url sem padding (22 caracteres)
  service  = gramática da FDD 15 (sem ".")
  instance = 1 a 253 bytes de 0x21 a 0x7E (pode conter ".")
  mac    = base64url sem padding (43 caracteres) de
           HMAC-SHA256(chave, "n3x.core/session/v1" 0x00 || texto antes do último ".")
  authorization = "Bearer " token
  ```
  ```go
  package sessiontoken // github.com/giustech/n3x-plugin-core/internal/sessiontoken

  const (
      Version      = "v1"
      Scheme       = "Bearer"
      KeyBytes     = 32
      NonceBytes   = 16
      NonceLen     = 22   // base64url de 16 bytes
      MACLen       = 43   // base64url de 32 bytes
      KIDLen       = 8
      MaxLen       = 1024 // igual a corev1.MaxTokenLen (teste de igualdade)
      MaxClockSkew = 2 * time.Minute
      MinLifetime  = 5 * time.Minute // igual ao mínimo de session.ttl (FDD 36)
      MaxLifetime  = 24 * time.Hour  // igual ao máximo de session.ttl (FDD 36)
  )

  // Claims aponta para dentro do texto do token: nenhuma cópia.
  type Claims struct {
      KID                 string
      IssuedAt, ExpiresAt int64 // segundos Unix
      Nonce, Service, Instance string
  }

  type Key struct{ /* 32 bytes, kid, lista livre de estados de HMAC */ }

  func NewKey(k [KeyBytes]byte) *Key
  func (k *Key) KID() string
  // Sign acrescenta o token a dst. Entradas já validadas pelo chamador.
  func (k *Key) Sign(dst []byte, c Claims) []byte
  // Verify confere forma, versão, kid, MAC e validade; 0 alocações no sucesso.
  func (k *Key) Verify(token string, now int64) (Claims, error)
  // Inspect lê os campos sem conferir o MAC (só para o sidecar agendar a renovação).
  func Inspect(token string) (Claims, error)
  // FromHeader tira o prefixo "Bearer " (esquema sem diferenciar caixa).
  func FromHeader(v string) (string, bool)
  func Header(token string) string

  var (
      ErrMalformed, ErrVersion, ErrKey, ErrSignature,
      ErrLifetime, ErrNotYetValid, ErrExpired error // Reason(): malformed, version, key, ...
  )
  ```
- Método: n/a (formato de texto no metadado HTTP/2 `authorization`)
- Semântica de status/headers:
  - **Por que texto e não binário em base64** (hipótese, H-01): os campos em texto deixam o `Authenticate` devolver `Service`, `Instance` e `Session` como pedaços do próprio token, sem alocar; a gramática garante leitura única (os campos à esquerda não têm `.`, o MAC tem tamanho fixo à direita e a instância é o meio), então dois tuplos diferentes nunca dão o mesmo texto assinado
  - **Domínio separado:** o MAC e o `kid` usam rótulos diferentes terminados em `0x00`; o texto assinado sempre começa com `v1.`, então nunca coincide com o rótulo do `kid`
  - **Base64 estrito** (`RawURLEncoding.Strict()`): bits sobrando no último caractere são recusados, para um token aceito ter uma única forma
  - **Tamanho:** no máximo 3+8+1+12+1+12+1+22+1+63+1+253+1+43 = 422 bytes com `iat` e `exp` de até 12 dígitos (418 com 10 dígitos, até o ano 2286); o metadado, no máximo 429 bytes; bem abaixo do limite de cabeçalhos de 16 KiB da FDD 33. `Verify` e `Inspect` recusam acima de `MaxLen` antes de qualquer HMAC
  - **Relógio** (hipótese, H-06): `iat` e `exp` vêm do relógio de parede do controller, em segundos Unix; `Verify` aceita `iat` até `MaxClockSkew` (2 min) no futuro e nenhum segundo depois de `exp`; `exp - iat` fora de `MinLifetime` a `MaxLifetime` é recusado
  - **Versão:** só `v1` na major 1 do core; formato novo entra como `v2` aceito ao lado do `v1` numa minor, com o sidecar caindo no caminho opaco (contrato 8) se não o entender
  - **Rotação:** o `kid` só diagnostica na v1 (uma chave por processo); aceitar a chave anterior pelo `kid` fica fora da v1
  - **Esquema `Bearer`** (hipótese, H-02): o servidor compara o esquema sem diferenciar caixa (`Bearer`, `bearer`); outro esquema, ou valor sem esquema, é `malformed`; o sidecar sempre envia `Bearer`
  - **Vetores:** `internal/sessiontoken/testdata/vectors.json` (`format: 1`) com a chave de teste `00` a `1f` (a mesma da `secret/secrettest`, FDD 03), nonces fixos e os tokens esperados, gerados uma vez na implementação e congelados; mudar um byte do formato muda o golden e exige `v2`

**Exemplo de requisição**
```go
k := sessiontoken.NewKey(testKey) // bytes 00..1f
tok := k.Sign(nil, sessiontoken.Claims{
    IssuedAt: 1792591201, ExpiresAt: 1792594801,
    Nonce: "q3Jx9kLm2Vb8Tz1Rw5Yc0A", Service: "pedidos", Instance: "pedidos-7d9f8b-xk2lp",
})
c, err := k.Verify(string(tok), 1792591300)
_, err2 := k.Verify(string(tok), 1792594801)
_, err3 := sessiontoken.NewKey(outraChave).Verify(string(tok), 1792591300)
```

**Exemplo de resposta**
```json
{
  "authorization": "Bearer v1.<kid da chave de teste>.1792591201.1792594801.q3Jx9kLm2Vb8Tz1Rw5Yc0A.pedidos.pedidos-7d9f8b-xk2lp.<mac do vetor 1>",
  "len(tok)": 129,
  "c": { "Service": "pedidos", "Instance": "pedidos-7d9f8b-xk2lp", "Nonce": "q3Jx9kLm2Vb8Tz1Rw5Yc0A", "IssuedAt": 1792591201, "ExpiresAt": 1792594801 },
  "err": null,
  "err2": "token expirado [expired]",
  "err3": "token de outra chave HMAC [key]"
}
```

---

**Contrato 3: `Authenticate` do core (implementação de `ctrl.Authenticator`, FDD 09, contrato 11)**
- Tipo: method (chamada local pelo interceptor de autenticação do servidor `17443`, FDD 33)
- Assinatura/Rota:
  ```go
  func (c *Core) Authenticate(ctx context.Context, token string) (ctrl.Identity, error) // delega a c.auth
  func (a *Authority) Authenticate(ctx context.Context, header string) (ctrl.Identity, error)
  ```
- Método: chamada local, em toda chamada fora da tabela pública
- Semântica de status/headers:
  - **Entrada:** o primeiro valor de `authorization`, como chegou (`Bearer v1...`); vazio o host já recusou com `missing_token`
  - **Sucesso:** `Identity{Role: RoleSidecar, Service, Instance, Session}` com `Valid()` verdadeiro; `Session` é o nonce (22 caracteres), identificador não secreto que a FDD 43 usa como `SessionRef.Nonce` (FDD 41). As três strings apontam para dentro do valor recebido: quem guarda por muito tempo copia (o Store da FDD 41 copia na gravação)
  - **Erros:** valores de pacote que embrulham `ctrl.ErrUnauthenticated` (`errors.Is` verdadeiro) e a sentinela do `sessiontoken`; `Error()` é um texto fixo em pt-BR sem token, sem `kid` recebido e sem nonce; `ctx.Err()` volta como está. Mapeamento no host (FDD 33): `ctx.Err()` dá `timeout`; os demais dão `UNAUTHENTICATED` "sessão ausente, inválida ou expirada", `result="unauthenticated"` e o texto abaixo no `detail` do `auth_failed`

| Motivo (`Reason()`) | Texto (`detail` no log do host) | Quando |
| --- | --- | --- |
| `not_ready` | chave HMAC ainda não carregada | antes do `Start` (defesa) |
| `malformed` | token malformado | sem `Bearer `, vazio, acima de 1.024 bytes, campo fora da forma |
| `version` | versão de token não suportada | versão diferente de `v1` |
| `key` | token de outra chave HMAC (controller reiniciado sem session.hmac_key ou chave trocada) | `kid` diferente |
| `signature` | assinatura do token inválida | MAC diferente |
| `lifetime` | validade do token fora da faixa | `exp - iat` fora de 5 min a 24 h |
| `not_yet_valid` | token emitido no futuro (relógio do controller voltou) | `iat` acima de agora mais 2 min |
| `expired` | token expirado | agora maior ou igual a `exp` |

  - **Sem I/O, sem log, sem trava global:** o estado de HMAC sai de uma lista livre de tamanho fixo (`2 x GOMAXPROCS`, no mínimo 8) feita no `Start`, sem `sync.Pool`, porque o coletor esvazia o `sync.Pool` e isso quebraria a medida de 0 alocações (hipótese, H-05); lista vazia cria um estado novo e conta como alocação rara; o índice de sessões é lido sob trava de leitura e só escrito na primeira vez de cada nonce
  - **Custo:** 0 alocações com token válido e nonce já visto; p99 até 20 µs; o contador `token_rejections_total{reason}` usa séries resolvidas no `New` (0 alocações)
  - **Tempo constante:** só o MAC é comparado com `hmac.Equal`; forma, versão e `kid` não são segredo, então a recusa rápida por eles não vaza nada sobre a chave

**Exemplo de requisição**
```text
:path /n3x.plugins.n3x.core.v1.Core/Register
authorization: Bearer v1.3f9a0c1e.1792591201.1792594801.q3Jx9kLm2Vb8Tz1Rw5Yc0A.pedidos.pedidos-7d9f8b-xk2lp.kV2xQ7nP0aZ3cR8tW5yB1mL6dF9hJ4sG2uE7oI0rT3w
x-request-id: 8b0f3c2e-4f1a-4c55-9a77-2d6e1b9f0c31
```

**Exemplo de resposta**
```json
{
  "Authenticate": { "Role": "sidecar", "Service": "pedidos", "Instance": "pedidos-7d9f8b-xk2lp", "Session": "q3Jx9kLm2Vb8Tz1Rw5Yc0A" },
  "token de outra chave": {
    "status": "rpc error: code = Unauthenticated desc = sessão ausente, inválida ou expirada",
    "log do host": "{\"level\":\"info\",\"component\":\"grpc\",\"msg\":\"auth_failed\",\"reason\":\"unauthenticated\",\"method\":\"/n3x.plugins.n3x.core.v1.Core/Heartbeat\",\"ip\":\"10.42.3.17\",\"detail\":\"token de outra chave HMAC (controller reiniciado sem session.hmac_key ou chave trocada)\",\"request_id\":\"...\",\"count\":1}"
  },
  "AllocsPerRun (token válido)": 0
}
```

---

**Contrato 4: `controller/auth` (`Authority`, handler `Login`, índice de sessões e `ValidateSession`)**
- Tipo: sdk (API Go interna da metade de controller do core)
- Assinatura/Rota:
  ```go
  package auth // github.com/giustech/n3x-plugin-core/controller/auth

  const (
      MaxIndexedSessions = 16384       // nonces lembrados para a FDD 43 (hipótese)
      IndexSweepInterval = time.Minute // varredura preguiçosa dos vencidos
      MinFreeStates      = 8
  )

  type Options struct {
      Session types.SessionConfig  // seção já validada (FDD 36 e esta FDD)
      Metrics *telemetry.Metrics   // env.Metrics(); nil em teste de unidade
      Log     *slog.Logger         // env.Log(); nil descarta
      Now     func() time.Time     // nil = time.Now (relógio de parede)
      Rand    io.Reader            // nil = crypto/rand.Reader (chave aleatória e nonce)
  }

  // New roda no controller.New: decodifica a chave configurada, loga os avisos e cria as métricas.
  func New(o Options) (*Authority, error)
  // Start roda no controller.Start: gera a chave aleatória quando não há hmac_key e publica a autoridade.
  func (a *Authority) Start() error
  // LoginServer devolve o handler da tabela pública (embute corev1.UnimplementedLoginServer).
  func (a *Authority) LoginServer(cenv ctrl.CoreEnv) corev1.LoginServer
  func (a *Authority) Authenticate(ctx context.Context, header string) (ctrl.Identity, error)

  // Session devolve emitido em e expira em de um nonce já emitido ou já autenticado (FDD 43).
  func (a *Authority) Session(nonce string) (SessionInfo, bool)
  type SessionInfo struct{ IssuedAt, ExpiresAt time.Time }

  // ValidateSession aplica as regras da seção session (Config.Validate, FDD 40).
  func ValidateSession(s types.SessionConfig) error

  var ErrInvalidKey = errors.New("auth: session.hmac_key inválida")
  ```
- Método: chamadas locais
- Semântica de status/headers:
  - **Chave:** com `hmac_key`, o texto aberto, sem espaços ASCII nas pontas, decodificado em base64 padrão estrito, tem de dar exatamente 32 bytes (a saída do `n3x-controller keygen`, FDD 30); sem `hmac_key`, 32 bytes de `crypto/rand` no `Start` (ADR-004). Os bytes ficam num `[32]byte` da autoridade; o texto decodificado intermediário é zerado por melhor esforço. A chave nunca aparece em log, erro, métrica nem no painel
  - **`Start` antes de tudo:** a ordem da FDD 08 (`New`, conferência, `Start`, `GRPC`, `PublicGRPC`) garante a chave antes do primeiro `Login`; o host liga o `Authenticate` só depois de todos subirem
  - **Índice de sessões** (hipótese, H-12): `map[[16]byte]{iat, exp}` sob `RWMutex`, alimentado na emissão e na primeira autenticação de cada nonce (cobre o restart com a mesma chave); até `MaxIndexedSessions` entradas; vencidos saem numa varredura preguiçosa no máximo a cada `IndexSweepInterval`; cheio só de válidos, o nonce novo não entra e o log `session_index_full` sai no máximo 1 por minuto; cerca de 1 MiB no pior caso, fora do Store e de `memory_limit`, como o diretório da FDD 40. Não guarda token, serviço nem instância
  - **`Session(nonce)`:** decodifica o nonce para `[16]byte` sem alocar; `false` para nonce desconhecido, malformado ou vencido; a FDD 43 usa para preencher `SessionRecord.IssuedAt` e `ExpiresAt` (FDD 41) e para encerrar streams cuja sessão venceu
  - **`ValidateSession`:** pura, sem log; caminhos relativos à seção (`session.hmac_key`), prefixados pelo host com `plugins["n3x.core"]` (FDD 36); `Detail` sem valor; o envelope ainda selado (validação com `--skip key`) é reconhecido pela mesma regra da FDD 40 (texto sem espaços nas pontas começando com `ENC[`, sem diferenciar caixa) e a regra é pulada
  - **Composição:** `Config.Validate()` é `joinFieldErrors(validateSession(c.Session), directory.ValidateUsers(c.Users))`, com `validateSession` delegando a `auth.ValidateSession` (FDD 40)

| Caminho | `Rule` | Condição | Texto (`Detail`) |
| --- | --- | --- | --- |
| `session.hmac_key` | `hmac_key_format` | não vazia, não é envelope e não é base64 padrão de exatamente 32 bytes | a chave HMAC precisa ser 32 bytes em base64 padrão com padding (gere com n3x-controller keygen e cifre com encrypt --plugin n3x.core) |
| `session.ttl` | `min`, `max`, `format` | faixa de 5 min a 24 h | regras do schema (FDD 36), antes da `Validate` |

**Exemplo de requisição** (`n3x-controller validate`)
```yaml
plugins:
  n3x.core:
    session:
      ttl: 1h
      hmac_key: "n3x-canary-chave-curta"
```

**Exemplo de resposta**
```text
plugins["n3x.core"].session.hmac_key: a chave HMAC precisa ser 32 bytes em base64 padrão com padding (gere com n3x-controller keygen e cifre com encrypt --plugin n3x.core) [hmac_key_format]
```

**Exemplo de requisição** (FDD 43 no handler `Register`)
```go
id := ctrl.Caller(ctx)
info, ok := c.auth.Session(id.Session)
```

**Exemplo de resposta**
```json
{ "ok": true, "info": { "IssuedAt": "2026-10-21T14:00:01Z", "ExpiresAt": "2026-10-21T15:00:01Z" } }
```

---

**Contrato 5: seção `plugins["n3x.core"].session` e a ligação na metade de controller**
- Tipo: config (seção do core no `n3x.yaml` do controller; tipos da FDD 36) e capacidades do core (FDD 09)
- Assinatura/Rota:
  ```yaml
  plugins:
    n3x.core:
      session:
        ttl: 1h                 # 5m a 24h; padrão 1h (FDD 36)
        hmac_key: ENC[v1:...]   # opcional; base64 de 32 bytes do keygen, cifrado com encrypt --plugin n3x.core
  ```
  ```go
  package controller // github.com/giustech/n3x-plugin-core/controller

  func New(cfg Config, env ctrl.Env) (ctrl.Instance, error) // auth.New(...) além dos componentes das FDDs 40 a 45
  func (c *Core) Start(ctx context.Context) error           // c.auth.Start(), além do resto
  func (c *Core) PublicGRPC(r ctrl.GRPCRegistrar)           // corev1.RegisterLoginServer(r, c.auth.LoginServer(c.env))
  func (c *Core) Authenticate(ctx context.Context, token string) (ctrl.Identity, error)
  func validateSession(s types.SessionConfig) error         // auth.ValidateSession(s)
  ```
- Método: n/a
- Semântica de status/headers:
  - **`ttl`:** validade de cada token, em segundos inteiros (fração de segundo ignorada); mudar exige restart; tokens já emitidos valem pelo `exp` que levam
  - **`hmac_key` ausente ou vazia** (`IsZero()`, H-10 da FDD 36): chave aleatória a cada subida; todo restart invalida as sessões e gera um novo login de cada sidecar; aviso `hmac_key_missing` (warn) uma vez por processo no `New`; `n3x_n3x_core_hmac_key_configured` igual a 0
  - **`hmac_key` em claro** (`InClear()`): aceita (L-64), aviso `hmac_key_in_clear` (warn) com `path`, uma vez no `New`; recomendado `ENC[...]` com a AAD `n3x:v1:controller:n3x.core` (FDD 03, L-05)
  - **`hmac_key` em `ENC[...]`** (`Sealed()`): caminho recomendado em produção; sobrevive ao restart, então um restart não gera login (ADR-004)
  - **Rotação:** trocar a `hmac_key` e reiniciar o controller força exatamente um novo login de cada sidecar (motivo `key` no log do host); não há janela com as duas chaves na v1
  - **Revogação:** não há armazenamento; uma sessão vale até o `exp`. Para cortar todas: trocar a chave e reiniciar. Para cortar um serviço: trocar o `secret_hash` dele no `n3x.yaml` do host e reiniciar com chave nova, porque, com a mesma chave, os tokens já emitidos valem até o `exp` (ADR-003 do `n3x-sdk`, "sem revogação de uma sessão específica")
  - **Dimensionamento:** com `verified_cache_ttl` (FDD 31) maior ou igual a 0,88 x `ttl`, a renovação de um pod que fez bcrypt no próprio login acerta o cache; o guia da FDD 47 recomenda manter os dois em 1 h

**Exemplo de requisição**
```yaml
plugins:
  n3x.core:
    session:
      ttl: 90m
```

**Exemplo de resposta** (logs do `New` e do `Start`)
```text
{"ts":"2026-10-21T14:00:00.100Z","level":"warn","component":"plugin","msg":"hmac_key_missing","plugin_id":"n3x.core","detail":"sem session.hmac_key: chave aleatória a cada subida; todo restart força novo login dos sidecars"}
{"ts":"2026-10-21T14:00:00.180Z","level":"info","component":"plugin","msg":"session_authority_ready","plugin_id":"n3x.core","key_source":"random","kid":"7c21e04b","ttl":"1h30m0s"}
```

---

**Contrato 6: `agent/session.Manager` (guarda do token, laço de login e gatilho da FDD 42)**
- Tipo: sdk (API Go interna da metade de proxy do core)
- Assinatura/Rota:
  ```go
  package session // github.com/giustech/n3x-plugin-core/agent/session

  type Options struct {
      Conn     grpc.ClientConnInterface  // env.Controller() da instância core
      Identity proxy.SidecarIdentity     // Service e Instance (FDD 09, contrato 5)
      Settings proxy.ControllerSettings  // Secret e LoginTimeout
      Reporter proxy.SessionReporter     // Sidecar().Session() (FDD 09, contrato 9)
      Metrics  *telemetry.Metrics        // env.Metrics()
      Log      *slog.Logger              // env.Log()
      Rand     func() float64            // nil = math/rand/v2 (jitter; não é segurança)
  }

  func New(o Options) (*Manager, error)
  // Run é a tarefa de fundo "session": login, renovação e novo login até o ctx terminar.
  func (m *Manager) Run(ctx context.Context)
  // Interceptors devolve o par entregue por ConnInterceptors() (contrato 7).
  func (m *Manager) Interceptors() (grpc.UnaryClientInterceptor, grpc.StreamClientInterceptor)
  // Current devolve a sessão ativa, sem o token.
  func (m *Manager) Current() (Info, bool)
  // Next espera uma sessão com geração maior que after (FDD 42).
  func (m *Manager) Next(ctx context.Context, after uint64) (Info, error)

  type Kind uint8
  const (
      KindFirst   Kind = iota + 1 // "first": primeiro login do processo
      KindRenewed                 // "renewed": renovação com o token anterior ainda válido
      KindRelogin                 // "relogin": novo login depois de recusa ou expiração
  )

  type Info struct {
      Generation uint64        // 1, 2, 3... a cada login bem-sucedido
      Kind       Kind
      IssuedAt   time.Time     // iat do token (relógio do controller); zero no caminho opaco
      ExpiresAt  time.Time     // LoginResponse.expires_at
      Lifetime   time.Duration // exp - iat
  }

  var ErrStopped = errors.New("session: gerenciador parado")
  ```
  ```go
  package agent // github.com/giustech/n3x-plugin-core/agent

  func (c *Core) ConnInterceptors() (grpc.UnaryClientInterceptor, grpc.StreamClientInterceptor) {
      if c.session == nil { // standalone: nunca chamado pelo executor; par de passagem por segurança
          return passUnary, passStream
      }
      return c.session.Interceptors()
  }
  ```
- Método: n/a (chamadas locais; uma instância por processo, criada em `agent.New`)
- Semântica de status/headers:
  - **Guarda do token (L-09):** `atomic.Pointer[state]` privado, com geração, cabeçalho `Bearer <token>` pronto, `kind`, `iat`, `exp` e a condição (ativo, renovando, recusado, expirado). Nunca em `memory.Map` (que descartaria em silêncio, FDD 04) nem em log; leitura pelo interceptor com uma carga atômica
  - **`Next`:** devolve na hora se a geração atual é maior que `after`; senão espera até a próxima sessão ou o fim do `ctx` (`ctx.Err()`); depois do `Run` terminar, `ErrStopped`. A FDD 42 guarda a última geração registrada e chama `Next(ctx, ultima)` em laço: cada sessão nova (primeira, renovação ou novo login) acorda o registrador uma vez
  - **`Current`:** `false` sem sessão ativa ou depois da expiração local; nunca devolve o token
  - **Segredo:** `Settings.Secret.Reveal()` só no momento de montar o `LoginRequest`; a mensagem não é logada nem guardada depois da chamada
  - **Uma instância:** o executor cria uma instância `core` por processo (FDD 09, L-10); o `Manager` vive o processo inteiro e sobrevive a reinícios da tarefa por panic

**Exemplo de requisição** (FDD 42, registrador)
```go
var last uint64
for {
    s, err := c.session.Next(ctx, last)
    if err != nil {
        return
    }
    last = s.Generation
    c.register(ctx, s) // Register; com KindRenewed, reabre também o heartbeat
}
```

**Exemplo de resposta**
```json
[
  { "Generation": 1, "Kind": "first", "IssuedAt": "2026-10-21T14:00:01Z", "ExpiresAt": "2026-10-21T15:00:01Z", "Lifetime": "1h0m0s" },
  { "Generation": 2, "Kind": "renewed", "IssuedAt": "2026-10-21T14:48:13Z", "ExpiresAt": "2026-10-21T15:48:13Z", "Lifetime": "1h0m0s" },
  { "Generation": 3, "Kind": "relogin", "IssuedAt": "2026-10-21T15:02:40Z", "ExpiresAt": "2026-10-21T16:02:40Z", "Lifetime": "1h0m0s" }
]
```

---

**Contrato 7: interceptors do core (`ConnInterceptors()`, unário e de stream)**
- Tipo: interceptors de cliente gRPC (FDD 09, contrato 3), os mais internos da conexão compartilhada (FDD 38, contrato 6)
- Assinatura/Rota:
  ```go
  const (
      RejectedMessage   = "sessão ausente, inválida ou expirada" // igual a ctrl.ErrUnauthenticated (teste de igualdade)
      NoSessionMessage  = "controller: sessão não estabelecida"
  )
  ```
- Método: toda chamada de qualquer plugin ao controller, depois da visão do `Env` e da guarda de panic
- Semântica de status/headers:
  - **Login:** o método `corev1.Login_Login_FullMethodName` passa sem token e sem espera
  - **Com token válido:** sobrescreve `authorization` (um valor) num `ctx` derivado; o `ctx` do plugin não muda
  - **Sem token, login em andamento:** espera dentro do prazo do `ctx` (que a visão já limitou a 5 s); vencido o prazo, `UNAVAILABLE` `NoSessionMessage`; cancelado, `CANCELED`
  - **Sem token, sem login em andamento:** `UNAVAILABLE` `NoSessionMessage` na hora, sem rede
  - **Recusa:** resposta `UNAUTHENTICATED` com `RejectedMessage` marca a geração usada como recusada, uma vez; `UNAUTHENTICATED` com outra mensagem (por exemplo, um handler de plugin que devolve esse código) não mexe na sessão. Em stream, vale o erro de `Header()` e de `RecvMsg`
  - **Sem teto próprio:** usa o prazo que recebe (L-08); não repete chamadas; não loga metadados nem o token
  - **Buffer:** os envios com `proxy.Buffered()` passam pelo mesmo interceptor; `UNAVAILABLE` e `UNAUTHENTICATED` são transitórios para o buffer (FDD 38), e a recusa vista num envio também dispara o novo login (P-01 da FDD 38)
  - **Custo:** uma carga atômica e um `ctx` com metadados por chamada (o lado cliente não tem meta de 0 alocações)

**Exemplo de requisição**
```go
// giustech.ratelimit, numa tarefa de fundo, em três momentos
_, err1 := cli.Report(ctx, req) // sessão ativa
_, err2 := cli.Report(ctx, req) // controller fora do ar, laço esperando backoff
_, err3 := cli.Report(ctx, req) // controller reiniciado sem hmac_key, token antigo
```

**Exemplo de resposta**
```json
{
  "err1": null,
  "metadados no servidor (err1)": { "authorization": "Bearer v1.3f9a0c1e....", "x-request-id": "1d5c8e0a-3b7f-4a2e-9c41-6f0d2b8a7e13" },
  "err2": "rpc error: code = Unavailable desc = controller: sessão não estabelecida",
  "err3": "rpc error: code = Unauthenticated desc = sessão ausente, inválida ou expirada",
  "depois de err3": "geração 1 recusada; Login em até 1 s; geração 2; Next acorda a FDD 42",
  "log": "{\"level\":\"warn\",\"component\":\"core\",\"msg\":\"session_rejected\",\"plugin_id\":\"n3x.core\",\"instance\":\"core\",\"method\":\"/n3x.plugins.giustech.ratelimit.v1.RateLimit/Report\",\"generation\":1,\"count\":1}"
}
```

---

**Contrato 8: tempos do sidecar (renovação, backoff e expiração local)**
- Tipo: sdk (constantes e regras do `agent/session`)
- Assinatura/Rota:
  ```go
  const (
      RenewFraction    = 0.8              // ADR-005
      RenewJitter      = 0.1              // ADR-005: 0,8 x L x (1 +- 0,1)
      RetryBase        = time.Second      // HLD do core: backoff de 1 a 60 s
      RetryMax         = 60 * time.Second
      RetryMultiplier  = 1.6              // o mesmo da conexão (FDD 38)
      RetryJitter      = 0.2              // HLD do core: jitter de 20%
      RejectSpread     = time.Second      // novo login entre 0 e 1 s depois da recusa (P-03 da FDD 31)
      FreshWindow      = 30 * time.Second // token recusado nessa janela conta como "recém-emitido" (hipótese)
      FreshRejectLimit = 3                // recusas seguidas de recém-emitidos antes do backoff completo (hipótese)
      OpaqueMinLife    = time.Minute      // caminho opaco: L limitado a 1 min .. (hipótese)
      OpaqueMaxLife    = 24 * time.Hour   // .. 24 h
  )
  ```
- Método: n/a
- Semântica de status/headers:
  - **Duração da sessão:** `L = exp - iat`, lidos do próprio token por `sessiontoken.Inspect` (os dois do relógio do controller), então a diferença de relógio entre sidecar e controller não muda o agendamento (hipótese, H-07); `expires_at` e `exp` diferentes por mais de 1 s: vale o menor e sai `session_response_invalid` (warn) uma vez
  - **Renovação:** `0,8 x L x (1 + u)`, `u` uniforme em [-0,1; +0,1], descontado o tempo real que o login levou; com 1 h, entre 43,2 e 52,8 min (hipótese de leitura do "jitter de 10%", H-08)
  - **Expiração local:** `L` descontado o tempo do login; o token deixa de ser usado nesse instante mesmo sem resposta do controller
  - **Backoff de falha:** espera `n` (a partir de 1) `= min(60 s, 1 s x 1,6^(n-1)) x (1 + v)`, `v` uniforme em [-0,2; +0,2], nunca abaixo de 1 s: cerca de 1; 1,6; 2,6; 4,1; 6,6; 10,5; 16,8; 26,8; 42,9; 60 s. `FAILED_PRECONDITION` e `UNIMPLEMENTED` pulam direto para 60 s. Sucesso zera `n`
  - **Novo login depois de recusa:** espera uniforme em [0; `RejectSpread`]; com `FreshRejectLimit` recusas seguidas de tokens recém-emitidos, passa ao backoff completo
  - **Primeiro login:** na hora, sem jitter (as subidas dos pods já se espalham)
  - **Relógio:** as esperas usam `proxy.Sleep` com um `ctx` derivado que a recusa cancela, para o kit controlar o tempo (FDD 06); a medida do tempo do login usa o relógio monotônico real

| Retorno do `Login` | `LoginResult` (FDD 09) | Próxima espera | Log |
| --- | --- | --- | --- |
| `OK` | `LoginOK` | renovação | `session_login_ok` (info) |
| `UNAUTHENTICATED` | `LoginFail` | backoff | `session_login_failed` `reason=credential` |
| `RESOURCE_EXHAUSTED` | `LoginFail` | backoff (primeira de cerca de 1 s) | `session_login_failed` `reason=limited` |
| `UNAVAILABLE`, `DEADLINE_EXCEEDED` | `LoginError` | backoff | `session_login_failed` `reason=unavailable` ou `timeout` |
| `FAILED_PRECONDITION`, `UNIMPLEMENTED` | `LoginError` | 60 s | `session_login_failed` `reason=protocol` |
| `INVALID_ARGUMENT`, `INTERNAL`, outro | `LoginError` | backoff | `session_login_failed` `reason=other` |
| `CANCELED` pelo próprio desligamento | não conta | fim do laço | `session_stopped` (info) |
| resposta inválida | `LoginError` | backoff | `session_response_invalid` (error) |

  - **`SessionReporter`:** `LoginAttempt` em toda tentativa que terminou (menos o cancelamento do desligamento); `SessionStarted(iat)` em cada sessão nova (no caminho opaco, a hora local da resposta); `SessionEnded()` na recusa, na expiração local e no fim do laço

**Exemplo de requisição** (relógio falso, `Rand` fixo em 0,5, TTL de 1 h, controller fora do ar das 14:10 às 14:20 e de volta com a mesma chave)
```json
{ "login": "14:00:00", "ttl": "1h", "u": 0.0, "queda": "14:10 a 14:20", "chave": "a mesma" }
```

**Exemplo de resposta**
```json
{
  "renovação": "14:48:00",
  "logins entre 14:00:01 e 14:47:59": 0,
  "n3x_controller_logins_total": { "ok": 2 },
  "n3x_controller_session_age_seconds às 14:47:00": 2820,
  "n3x_controller_session_age_seconds às 14:49:00": 60
}
```

---

### 6. Erros, exceções e fallback

**Matriz de erros previstos e tratamentos**

| Condição | Tratamento | Observações |
| --- | --- | --- |
| `LoginRequest` fora dos limites | `INVALID_ARGUMENT` com o nome do campo; sem `Verify`; `logins_total{result="invalid_request"}` | Não conta como falha de credencial (H-13) |
| Credencial errada | `UNAUTHENTICATED` "credencial inválida", devolvido como veio do `Verify` | Sidecar: `LoginFail`, backoff |
| IP ou serviço bloqueado, verificador ocupado | `RESOURCE_EXHAUSTED` como veio do `Verify` | Sidecar: `LoginFail`, primeira espera de cerca de 1 s |
| Prazo no `Verify` | `ctx.Err()` como veio; host mapeia para `DEADLINE_EXCEEDED` | `logins_total{result="timeout"}` |
| Autoridade não publicada, falha do gerador de nonce injetado | `UNAVAILABLE` "core: sessão indisponível" | Só em teste ou bug |
| Token sem `Bearer `, malformado ou acima de 1.024 bytes | `Authenticate` recusa `malformed` | Host: `UNAUTHENTICATED` genérico |
| Token de outra versão | `version` | Formato novo entra por minor ao lado do `v1` |
| Token de outra chave | `key`, com texto que aponta o restart sem `hmac_key` | Sidecar refaz o login uma vez |
| Token adulterado ou forjado | `signature` | Alerta sugerido; sem recusa diferente para o cliente |
| Token expirado | `expired` | Normal se a renovação falhou; o sidecar refaz o login |
| Token com `iat` no futuro além de 2 min | `not_yet_valid` | Relógio do controller voltou; um novo login resolve |
| Token com validade fora de 5 min a 24 h | `lifetime` | Só com chave reaproveitada de outro sistema ou bug |
| `ctx` vencido no `Authenticate` | `ctx.Err()`; host: `timeout` | Sem I/O, só a checagem no início |
| Token de outra instância do mesmo serviço apresentado por outro pod | Aceito pelo `Authenticate` (MAC válido); a identidade é a do token | A FDD 43 recusa `Register` com identidade diferente (`PERMISSION_DENIED`) e detecta conflito |
| Token de outro serviço | Aceito como do serviço do token; nunca vira outro serviço | Amarração do ADR-005 |
| `hmac_key` fora do formato | `hmac_key_format` na `Validate`; controller não sobe | Com `--skip key`, regra pulada para envelope |
| `hmac_key` ausente | Chave aleatória; `hmac_key_missing` | ADR-004 |
| `hmac_key` em claro | Aceita; `hmac_key_in_clear` | L-64 |
| Chave inválida no `New` (sem `Validate`) | `ErrInvalidKey`; o core não sobe | Defesa |
| Índice de sessões cheio | Nonce novo não entra; `Session(nonce)` dá `false`; `session_index_full` 1 por minuto | A FDD 43 usa a hora do `Register` como reserva |
| Sidecar sem sessão, login em andamento | Chamada espera dentro do prazo; depois `UNAVAILABLE` | FR-002 |
| Sidecar sem sessão, sem login em andamento | `UNAVAILABLE` na hora | Buffer guarda os `Buffered` |
| `UNAUTHENTICATED` genérico numa chamada com token | Geração recusada uma vez; novo login entre 0 e 1 s | N recusas, 1 login |
| `UNAUTHENTICATED` com outra mensagem | Passa ao plugin, sessão intocada | Handler de plugin com código errado não derruba a sessão |
| Recusas seguidas de tokens recém-emitidos | Backoff completo e `session_rejected_fresh` | Evita laço de login |
| Renovação falha | Token antigo até a expiração local; repete com backoff | Chamadas não percebem |
| Expiração local sem renovação | Sem token; `SessionEnded`; `session_expired`; laço segue | `session_invalidations_total{cause="expired"}` |
| Resposta de login inválida | `LoginError`; `session_response_invalid`; backoff | Controller com bug |
| Token opaco para o `Inspect` | Caminho opaco, `session_token_opaque` uma vez | Sensível ao relógio só nesse caso |
| Panic na tarefa `session` | Reinício pelo runtime; renovação imediata se havia sessão | Estado sobrevive |
| Panic no interceptor | `INTERNAL` pela guarda da FDD 38 | `n3x_plugin_panics_total{plugin="n3x.core"}` |
| Desligamento | Laço para; token mantido para o `Exit` em 1,5 s | `SessionEnded` |

**Estratégias de resiliência**
- **Timeouts:** login com `controller.timeouts.login` (10 s) no cliente, imposto pela visão e repetido pelo laço no próprio `ctx`; `plugin_call_timeout` (10 s) no servidor; `Authenticate` sem I/O e com o prazo da chamada; o interceptor não impõe teto (L-08)
- **Retries:** só o laço de login repete; o interceptor e o handler nunca repetem chamadas; o buffer da FDD 38 repete os envios `Buffered`
- **Backoff e jitter:** 1 s a 60 s, multiplicador 1,6, jitter de 20% nas falhas; 0 a 1 s depois de recusa; renovação com jitter de 10%. Com 500 sidecars, o restart sem chave espalha os novos logins no primeiro segundo e a deduplicação da FDD 31 junta os de um mesmo serviço
- **Circuit breaker:** não há; o equivalente é o backoff com teto de 60 s, o bloqueio por IP e por serviço do verificador e a escalada por recusas de tokens recém-emitidos
- **Estruturais:** token em campo privado (sem descarte); troca atômica sem trava no caminho de cada chamada; recusa tratada uma vez por geração; lista livre de estados de HMAC; índice limitado

**Política de fallback**
- Sem sessão: chamadas recebem `UNAVAILABLE` (ou esperam um login em andamento), envios `Buffered` esperam; o tráfego nunca é afetado (FR-002, ADR-008 do core)
- Sem `hmac_key`: chave aleatória e aviso; o sistema funciona, com um novo login por restart
- Sem formato de token conhecido no sidecar: agendamento pelo relógio local, com aviso
- Nunca há fallback de autenticação: na dúvida, o `Authenticate` recusa

**Invariantes**
- O token só existe no `session.Manager`, no `ctx` derivado do transporte e no fio
- `Authenticate` aceita só token com MAC conferido por `hmac.Equal` com a chave atual, dentro da validade
- Nenhuma recusa ou erro, nos dois lados, contém token, segredo, chave, `kid` recebido ou nonce
- Uma recusa de uma geração dispara no máximo um login
- O token antigo continua em uso até a troca atômica; nenhuma chamada espera a renovação
- O sidecar nunca envia token depois da expiração local
- O controller não guarda tokens; o índice guarda só nonce, emitido em e expira em
- `Session` da `Identity` é igual ao nonce do token e às strings do próprio valor recebido

---

### 7. Observabilidade

**Métricas** (as `n3x_n3x_core_*` são do core, prefixo do ID, FDD 01 e ADR-010; a FDD 45 consolida)

| Métrica | Tipo | Rótulos | Processo | Quando muda |
| --- | --- | --- | --- | --- |
| `n3x_n3x_core_logins_total` | counter | `result` (`ok`, `fail`, `blocked`, `busy`, `timeout`, `invalid_request`, `error`) | controller | cada chamada ao handler `Login` (ADR-010; valores são hipótese, H-14) |
| `n3x_n3x_core_token_rejections_total` (nova) | counter | `reason` (`malformed`, `version`, `key`, `signature`, `lifetime`, `not_yet_valid`, `expired`, `not_ready`) | controller | cada recusa do `Authenticate` (hipótese) |
| `n3x_n3x_core_hmac_key_configured` (nova) | gauge 0 ou 1 | | controller | no `New`: 1 com `hmac_key`, 0 com chave aleatória (hipótese) |
| `n3x_n3x_core_session_index_entries` (nova) | gauge | | controller | entradas do índice de sessões, calculadas na coleta (hipótese) |
| `n3x_controller_logins_total` | counter | `result` (`ok`, `fail`, `error`) | sidecar | `SessionReporter.LoginAttempt` (FDD 09, L-12; publicada pela FDD 16) |
| `n3x_controller_session_age_seconds` | gauge | | sidecar | `SessionStarted` e `SessionEnded` (FDD 09; publicada pela FDD 16) |
| `n3x_n3x_core_session_invalidations_total` (nova) | counter | `cause` (`rejected`, `expired`) | sidecar | sessão perdida por recusa do controller ou por expiração local (hipótese) |

- Cardinalidade constante: nenhum rótulo com serviço, instância, IP, nonce ou token; no sidecar, os rótulos automáticos `plugin="n3x.core"` e `instance="core"` (FDD 06)
- A latência do `Login` e do resto vem de `n3x_controller_grpc_duration_seconds{method}` do host (FDD 33); as autenticações por resultado, de `n3x_controller_auth_total{result}`; os bcrypt e bloqueios, das métricas da FDD 31
- `n3x_controller_logins_total` tem sentido diferente no sidecar (tentativas do core) e no controller (verificador, FDD 31), já registrado pela FDD 06

**Logs** (JSON pelo handler do SDK; `component=core` no sidecar e o valor do host no controller; nunca token, segredo, chave, nonce, `kid` recebido ou valor de `authorization`)

| Evento (`msg`) | Nível | Campos | Processo | Amostragem |
| --- | --- | --- | --- | --- |
| `hmac_key_missing` | warn | `plugin_id`, `detail` | controller | uma vez por processo (FDD 36) |
| `hmac_key_in_clear` | warn | `plugin_id`, `path` | controller | uma vez por processo (FDD 36) |
| `session_authority_ready` | info | `key_source` (`config`, `random`), `kid` (da chave atual), `ttl` | controller | uma vez, no `Start` |
| `session_issued` | debug | `service`, `instance`, `ttl`, `request_id` | controller | cada login |
| `login_invalid_request` | info | `field`, `request_id`, `count` | controller | no máximo 1 por segundo |
| `session_index_full` | warn | `entries`, `count` | controller | no máximo 1 por minuto |
| `session_login_ok` | info | `kind`, `generation`, `expires_at`, `lifetime_s`, `duration_ms`, `attempt` | sidecar | cada sessão nova |
| `session_login_failed` | warn | `kind`, `code`, `reason`, `attempt`, `next_retry_ms` | sidecar | primeira falha da sequência e depois no máximo 1 por minuto, com `count` |
| `session_rejected` | warn | `method`, `generation`, `count` | sidecar | no máximo 1 por minuto |
| `session_rejected_fresh` | warn | `count`, `next_retry_ms` | sidecar | no máximo 1 por minuto |
| `session_expired` | warn | `generation`, `lifetime_s` | sidecar | cada expiração local |
| `session_token_opaque` | warn | `lifetime_s` | sidecar | uma vez por processo |
| `session_response_invalid` | error | `problem` (`token_empty`, `token_size`, `token_charset`, `expires_missing`, `expiry_mismatch`) | sidecar | no máximo 1 por minuto |
| `session_stopped` | info | `generation`, `active` | sidecar | uma vez, no fim do laço |

- Exemplo (sidecar): `{"ts":"2026-10-21T14:00:01.212Z","level":"info","component":"core","service":"pedidos","msg":"session_login_ok","plugin_id":"n3x.core","instance":"core","kind":"first","generation":1,"expires_at":"2026-10-21T15:00:01Z","lifetime_s":3600,"duration_ms":212,"attempt":1}`
- Exemplo (sidecar): `{"ts":"2026-10-21T14:05:00.010Z","level":"warn","component":"core","service":"pedidos","msg":"session_login_failed","plugin_id":"n3x.core","instance":"core","kind":"first","code":"Unauthenticated","reason":"credential","attempt":1,"next_retry_ms":1040}`
- Identidade da instância do sidecar no campo `sidecar_instance` quando aparece (FDD 25), porque `instance` é a instância de plugin
- Correlação: o `Login` leva `x-request-id` (FDD 38); o log do host e o `session_issued` usam o mesmo `request_id` (L-13)

**Tracing**
- Sem tracing distribuído na v1 (ADR-010 do core, ADR-009 do SDK); correlação pelo `x-request-id` e pelo par serviço e instância. O nonce correlaciona sessão e inventário só dentro do controller (FDD 43), sem ir para o log

**Dashboards e alertas**
- Painel de referência (FDD 47): logins por resultado nos dois lados, idade da sessão, recusas de token por motivo, invalidações por causa
- Alertas sugeridos (a FDD 45 consolida):
  - `n3x_n3x_core_hmac_key_configured == 0` em produção (aviso: todo restart força login de todos)
  - `rate(n3x_n3x_core_token_rejections_total{reason="signature"}[5m]) > 0` (token adulterado ou forjado)
  - `rate(n3x_n3x_core_token_rejections_total{reason="not_yet_valid"}[5m]) > 0` (relógio do controller voltou)
  - `increase(n3x_n3x_core_token_rejections_total{reason="key"}[15m]) > 0` fora de uma janela de deploy (restart sem chave ou chave trocada)
  - `increase(n3x_n3x_core_logins_total{result="fail"}[15m]) > 3` e `increase(n3x_controller_logins_total{result="fail"}[15m]) > 3` (credencial recusada)
  - `n3x_controller_session_age_seconds > 3600` por 5 min (renovação falhando, FDD 09)
  - `rate(n3x_n3x_core_session_invalidations_total{cause="expired"}[15m]) > 0` (renovação não chegou a tempo)

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| Go | 1.26 | `crypto/rand` sem erro; `hmac.Equal`; `math/rand/v2` para jitter |
| `github.com/giustech/n3x-sdk` | a do `go.mod` do core (limite inferior da faixa `sdk`) | `plugin.Secret`, `NewSecret`; `proxy.CoreEnv`, `Sidecar`, `SessionReporter`, `Sleep`, `Background`; `ctrl.CoreEnv`, `Authenticator`, `Identity`, `ErrUnauthenticated`, sentinelas; `telemetry`; `sdktest`, `ctrltest`, `hostconformance` e `secret/secrettest` só em testes |
| `google.golang.org/grpc` e `google.golang.org/protobuf` | as do `go.mod` do SDK | interceptors de cliente, `status`, `metadata`, `WaitForReady`, `timestamppb` |
| `github.com/giustech/n3x-plugin-core/proto/n3x/plugins/n3x/core/v1` | FDD 36, `ProtoMajor = 1` | `LoginRequest`, `LoginResponse`, `RegisterLoginServer`, `Login_Login_FullMethodName`, `MaxTokenLen` |
| `n3x-sidecar` `core/grpcconn` | FDD 38 | teto do login, isenção por método, guarda de panic, buffer; teste de igualdade de `LoginMethod` com o stub (FDD 48) |
| `n3x-controller` `host/internal/credential`, `plugins`, `grpcsrv` | FDDs 31, 32 e 33 | `Verify`, `Host.Authenticate`, mapeamento de `Authenticate`, `CallerIP`, retirada de `authorization` |
| FDD 40 (`controller/directory`) | mesma sessão S27 | composição da `Validate` e regra de envelope |
| FDDs 42 e 43 | consumidoras | `Manager.Next` e `Authority.Session` |

**Garantias de compatibilidade**
- **Fio:** o proto não muda (FDD 36); o formato `v1` do token fica congelado na major 1 do core pelos vetores golden; formato novo entra como `v2` aceito junto do `v1` numa minor
- **Sidecar e controller com versões diferentes do core** (mesma major do proto): o sidecar trata o token como opaco quando não o entende (contrato 8) e continua; o controller só aceita os formatos que ele mesmo emite
- **Mensagens:** os textos genéricos do host (`RejectedMessage`) e das sentinelas (FDD 09) ficam estáveis na major do SDK; o core confere a igualdade com `ctrl.ErrUnauthenticated` num teste
- **Config:** `session.ttl` e `session.hmac_key` só crescem por adição (ADR-020); campo novo para a chave anterior (rotação sem restart) seria minor fora da v1
- **APIs internas** (`controller/auth`, `agent/session`, `internal/sessiontoken`): estáveis para as FDDs 42, 43 e 45 dentro do repositório; `internal/` não é importável de fora
- **Dependências:** `internal/sessiontoken` só importa biblioteca padrão; `agent/...` não alcança `controller/` do SDK (`go list -deps`, FDD 36)

---

### 9. Critérios de aceite técnicos

- **Vetores do formato:** `internal/sessiontoken/testdata/vectors.json` com no mínimo 12 casos (válido, instância com `.`, instância com 253 bytes, serviço com 63 bytes, cada um dos 7 motivos de recusa, base64 com bits sobrando) passa em `Sign`, `Verify` e `Inspect`; `Sign` dá os mesmos bytes em 100 execuções e nas CIs amd64 e arm64; `MaxLen == corev1.MaxTokenLen`
- **Fuzzing:** `FuzzVerify` e `FuzzInspect` por no mínimo 5 minutos cada na CI noturna, sem panic; todo token aceito por `Verify` tem `Sign` igual a ele (ida e volta)
- **Testes de segurança** (critério do PRD): credencial inválida dá `UNAUTHENTICATED` "credencial inválida" no `Login`; chamada sem token dá `UNAUTHENTICATED` sem chamar o core; token expirado dá `expired`; token com um byte trocado em qualquer posição (varredura de todas as posições de um token de 129 bytes) é recusado; token de outra chave dá `key`; token emitido para `pedidos/pedidos-0` apresentado num `Register` com identidade `pedidos/pedidos-1` passa no `Authenticate` com a identidade do token (a recusa `PERMISSION_DENIED` é critério da FDD 43); token com `iat` 3 min no futuro dá `not_yet_valid`; token com `exp - iat` de 25 h, assinado com a chave certa, dá `lifetime`
- **`Authenticate` (pedido das FDDs 09 e 12):** `sdktest.AssertZeroAllocsFunc(t, "authenticate", fn)` sobre `h.For("n3x.core").Instance().(ctrl.Authenticator)` com token obtido por `Login` no `bufconn` passa no passo sem `-race`; benchmark com 8 goroutines dá p99 de no máximo 20 µs e média de no máximo 5 µs no runner da CI; com `ctx` cancelado devolve `context.Canceled` sem ler o token; nenhum erro devolvido contém o token, o nonce ou o `kid` recebido (teste com 50 tokens inválidos e canário)
- **`Login`:** o handler, com `FakeCredentials` que aceita, custa no máximo 1 ms no p99 e no máximo 40 alocações por chamada; os 4 campos fora dos limites (serviço com maiúscula, serviço com 64 bytes, instância com espaço, segredo vazio) dão `INVALID_ARGUMENT` sem chamar o `Verify` (`FakeCredentials.Calls()` vazio); `ErrInvalidCredentials`, `ErrLoginBlocked`, `ErrVerifierBusy` e `context.DeadlineExceeded` do `Verify` saem com `errors.Is` verdadeiro e o `status.Code` da FDD 09; `plugin.NewSecret` recebe o segredo e nenhum log do core contém o canário do segredo
- **Renovação (relógio falso):** com TTL de 1 h e `Rand` fixo, a renovação acontece em 48 min com `u = 0`, em 43,2 min com `u = -0,1` e em 52,8 min com `u = +0,1`; com 1.000 sorteios, todos ficam entre 43,2 e 52,8 min; com o relógio de parede do processo do sidecar deslocado em mais ou menos 30 min (token com `iat` e `exp` do controller), o ponto não muda; com TTL de 5 min, a renovação fica entre 3 min 36 s e 4 min 24 s
- **Troca de token:** 1.000 chamadas concorrentes durante a renovação terminam todas com `OK` (as iniciadas antes da troca com o token antigo, as depois com o novo), sem nenhuma espera maior que 1 ms causada pela renovação
- **Recusa e novo login:** 100 chamadas recusadas com `UNAUTHENTICATED` genérico pela mesma geração geram exatamente 1 `Login`, iniciado entre 0 e 1 s depois da primeira recusa (relógio falso); `UNAUTHENTICATED` com outra mensagem gera 0 `Login`; 3 recusas seguidas de tokens recém-emitidos levam ao backoff completo e ao log `session_rejected_fresh`
- **Sem sessão:** com login em andamento, a chamada espera e segue com o token quando o login termina; com login falho, recebe `UNAVAILABLE` `NoSessionMessage`; sem login em andamento, recebe `UNAVAILABLE` em menos de 1 ms, sem nenhuma escrita no transporte; o método de login nunca leva `authorization`
- **Backoff:** com o controller recusando, as esperas seguem 1; 1,6; 2,56; 4,1; 6,55 s até 60 s com jitter dentro de mais ou menos 20% (relógio falso, 1.000 sorteios); `FAILED_PRECONDITION` espera 60 s mais ou menos 20%; sucesso zera a sequência
- **Restart do controller** (`ctrltest` com dois hosts em sequência ligados à mesma conexão de teste, ou dois `Authority` com a mesma chave em teste de unidade): com a mesma `hmac_key`, um token emitido pelo primeiro é aceito pelo segundo, `Session(nonce)` do segundo devolve `iat` e `exp` corretos depois da primeira autenticação e o sidecar não chama `Login`; com chave aleatória, o segundo recusa com `key`, o sidecar faz exatamente 1 `Login` e `Next` acorda com `KindRelogin`
- **Volta ao painel em até 30 s** (critério do PRD): medido de ponta a ponta na FDD 48 com e sem `hmac_key`; nesta FDD, o caminho do sidecar até o novo token, depois da primeira recusa, termina em no máximo 1 s mais a duração do `Login`
- **Conformidade:** `hostconformance.RunCoreEnv(t, ctrltest.Harness(), hostconformance.WithCore(controller.Plugin()))` passa inteiro no `n3x-plugin-core`, inclusive `authorization` retirado e identidade válida; `conformance.RunCore` da FDD 07 passa com o core real (interceptor o mais interno, `ConnInterceptors()` chamado uma vez); teste de duas metades com `ctrltest.Link` faz login, chama um serviço de teste e renova em 48 min de relógio falso
- **Validação:** `hmac_key` com 31 bytes, 33 bytes, base64url, sem padding ou com texto qualquer dá `hmac_key_format` com caminho `plugins["n3x.core"].session.hmac_key`; a saída do `keygen` passa, com e sem `\n` final; com `secret.FormatChecker()` (`--skip key`), um envelope passa sem problema; a ordem dos problemas é schema, sessão, usuários (exemplo conjunto com a FDD 40); canário em `hmac_key` com 0 ocorrência em `err.Error()`
- **Avisos:** sem `hmac_key`, `hmac_key_missing` sai exatamente uma vez e `n3x_n3x_core_hmac_key_configured` vale 0; em claro, `hmac_key_in_clear` uma vez; selada, nenhum dos dois
- **Varredura de segredos:** com o canário `n3x-canary-` como `controller.secret` e como `hmac_key`, e com o texto exato de cada token emitido no teste, `secrettest.Leaks` não acha nada nos logs, erros, status e métricas dos dois lados
- **Dependências e qualidade:** `go list -deps ./internal/sessiontoken` só lista biblioteca padrão; `go list -deps ./agent/...` não lista `n3x-sdk/controller`; `go vet` e `go test -race ./...` passam; cobertura de linhas de no mínimo 90% em `internal/sessiontoken`, `controller/auth` e `agent/session` (hipótese, igual às FDDs anteriores); a constante `skeleton` da versão mínima (FDD 36) sai de `agent` e de `controller` quando esta FDD e as 40 e 42 estiverem implementadas

---

### 10. Riscos e mitigação

### Bug no interceptor ou no laço de login corta todos os plugins do controller

- **Probabilidade:** média
- **Impacto:** nenhum plugin fala com o controller (painel vazio, eventos perdidos depois do buffer); o tráfego da aplicação não é afetado
- **Mitigação:**
    - TDD do `agent/session` com relógio falso, conformidade `RunCore` e teste de duas metades com `Link`
    - Guarda de panic da FDD 38 e reinício da tarefa pelo runtime, com o estado preservado
    - Alertas `n3x_controller_session_age_seconds > 3600` e de invalidações
- **Plano de contingência:** imagem de build com a versão anterior do core (FDD 46); o sidecar segue servindo tráfego até a troca

### Rajada de logins depois de restart sem `hmac_key`

- **Probabilidade:** alta onde a chave não está configurada
- **Impacto:** fila de bcrypt cheia, `RESOURCE_EXHAUSTED` em cascata, volta ao painel acima de 30 s com muitos serviços (I-09 da FDD 31: cerca de 100 serviços em 1 vCPU)
- **Mitigação:**
    - Espera de 0 a 1 s antes do novo login e backoff com jitter depois de `busy`
    - Deduplicação e cache por serviço da FDD 31 (um bcrypt por serviço)
    - Aviso `hmac_key_missing`, métrica `hmac_key_configured` e guia da FDD 47 recomendando `ENC[...]`
- **Plano de contingência:** configurar a `hmac_key`; aumentar `bcrypt_concurrency` com mais vCPU (FDD 51)

### Token vazado usado por outro processo até expirar

- **Probabilidade:** baixa (TLS 1.3, token só no core)
- **Impacto:** quem tem o token fala como aquela instância por até 1 h; não pode virar outro serviço
- **Mitigação:**
    - TTL de 1 h e renovação silenciosa (ADR-005)
    - Token fora de logs, erros, eventos e `memory.Map`, conferido por varredura
    - Conflito de identidade detectado pela FDD 43 (dois nonces ativos)
    - Encerramento dos streams na expiração da sessão (pedido à FDD 43)
- **Plano de contingência:** trocar a `hmac_key` e reiniciar o controller (revoga todas as sessões)

### Credencial de um serviço usada para declarar qualquer instância dele

- **Probabilidade:** média (por desenho: credencial por serviço)
- **Impacto:** um pod comprometido pode registrar instâncias falsas do próprio serviço e poluir o inventário daquele serviço
- **Mitigação:**
    - Amarração da identidade ao token e conflito de identidade (ADR-005, FDD 43)
    - Bloqueio por IP e por serviço no login (FDD 31)
- **Plano de contingência:** trocar o segredo do serviço (`credential new`) e a `hmac_key`; identidade por instância com mTLS fica para depois da v1

### Diferença de relógio quebra a sessão

- **Probabilidade:** baixa no sidecar (o agendamento usa `exp - iat`); média no controller sem NTP
- **Impacto:** renovação atrasada só no caminho opaco; no controller, relógio voltando gera `not_yet_valid` e um novo login de todos
- **Mitigação:**
    - Duração lida do próprio token, com o relógio monotônico do sidecar
    - Tolerância de 2 min para `iat` no futuro; alerta de `not_yet_valid`
    - Guia da FDD 47 com NTP no controller
- **Plano de contingência:** ajustar `MaxClockSkew` numa minor depois de medir

### Laço de login por recusa persistente

- **Probabilidade:** baixa
- **Impacto:** login a cada segundo, ruído no log e bcrypt desperdiçado se o cache não acerta
- **Mitigação:**
    - Uma recusa por geração; recusa só com a mensagem genérica do host
    - Escalada para backoff completo depois de 3 recusas de tokens recém-emitidos
- **Plano de contingência:** desligar o novo login automático por recusa numa versão de correção, mantendo só a expiração local

### Índice de sessões vira consumo de memória ou estado divergente

- **Probabilidade:** baixa
- **Impacto:** até cerca de 1 MiB; com o índice cheio, a FDD 43 perde o "emitido em" exato
- **Mitigação:**
    - Limite de 16.384 entradas, varredura preguiçosa, entrada só para token com MAC válido
    - Métrica `session_index_entries` e log `session_index_full`
- **Plano de contingência:** a FDD 43 usa a hora do primeiro `Register` como reserva; aumentar o limite numa minor

### Renovação lida como conflito de identidade

- **Probabilidade:** média sem coordenação entre as FDDs 42 e 43
- **Impacto:** a renovação troca o nonce; se a sessão antiga e a nova mandarem heartbeat no mesmo intervalo, o painel mostra conflito falso a cada 48 min
- **Mitigação:**
    - `Next` com `KindRenewed` para a FDD 42 reabrir o heartbeat com o token novo e fechar o antigo
    - Pedido à FDD 43: tratar a troca de nonce da mesma instância, com o stream antigo encerrado, como renovação e exigir o conflito por 2 intervalos seguidos
- **Plano de contingência:** a FDD 43 ignora o conflito quando o nonce antigo deixa de mandar heartbeat em até 1 intervalo

### Lista livre ou índice quebrando a meta de 0 alocações

- **Probabilidade:** média na primeira implementação
- **Impacto:** critério das FDDs 09 e 12 reprovado; custo extra com 500 sidecars
- **Mitigação:**
    - `AssertZeroAllocsFunc` no CI sem `-race` desde o primeiro PR
    - Sem `sync.Pool`, sem `strings.Split`, sem `strconv` com erro, sem conversão de `string` para `[]byte` no caminho comum
- **Plano de contingência:** aceitar 1 alocação documentada (estado de HMAC por chamada) e rever a meta da FDD 09 numa minor
