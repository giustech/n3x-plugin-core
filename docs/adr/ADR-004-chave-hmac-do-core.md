# ADR-004: Chave HMAC do core, aleatória ou `session.hmac_key`

Status: Aceito
Data: 2026-10-20

## Contexto
O core emite e valida o token de sessão, mas o HLD do controller previa que o host carregasse a chave (`keys.hmac_file`), e nenhum handle do SDK a entregava ao core. Além disso, se todo restart invalidar os tokens, todos os sidecars refazem o login ao mesmo tempo. O cache de credencial do host é por serviço, então o custo é de cerca de um bcrypt por serviço (cerca de 12 s de CPU com 50 serviços). Isso cabe na meta de 30 s, mas gera uma rajada de logins a cada deploy.

## Decisão
- A chave HMAC pertence ao core.
- Por padrão, o core gera 32 bytes de `crypto/rand` no `Start`.
- Opcionalmente, aceita `session.hmac_key: ENC[...]` na própria seção do `n3x.yaml`. Esse é o caminho recomendado em produção.
- Sem a chave configurada, a `Validate` emite um aviso no log.
- O `keys.hmac_file` sai do `n3x.yaml` do host do controller.
- Rotação: trocar no yaml e reiniciar o controller, o que força um único novo login de todos os sidecars.

## Justificativa
- Evita aumentar o contrato de capacidades privilegiadas do SDK.
- Com a chave fixa, os tokens sobrevivem ao restart, não há rajada de logins, e o inventário volta em cerca de 1 intervalo de heartbeat (10 s).
- A chave aleatória mantém o dev e o compose sem configuração extra.

## Trade-off
- Sem `hmac_key`, todo restart gera uma rajada de logins. A meta de 30 s continua valendo até cerca de 200 serviços (com 2 bcrypt simultâneos de cerca de 250 ms); acima disso, só com a chave configurada.
- Não há rotação sem restart na v1.

## Requisitos relacionados
FR-002, FR-005, FR-016; volta ao painel em até 30 s.
