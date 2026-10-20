# ADR-003: Usuários de leitura só pelo `n3x.yaml`

Status: Aceito
Data: 2026-10-20

## Contexto
O FR-009 previa que o root criasse, desativasse, removesse e redefinisse usuários de leitura pelo painel, e que cada usuário trocasse a própria senha. Com o armazenamento em memória (ADR-001), tudo isso sumiria a cada restart.

## Decisão
- Todos os usuários de leitura são declarados em `plugins.n3x.core.users` no `n3x.yaml` do controller, com `name`, `password_hash` (bcrypt de custo 12 ou mais, marcado como segredo no schema) e `disabled`.
- O diretório é só leitura:
  - `CheckUser` confere a senha com bcrypt
  - `ChangePassword` devolve "não suportado"
- A `Validate` falha a subida nestes casos:
  - nome duplicado
  - nome igual ao do root
  - hash que não é bcrypt, ou que tem custo menor que 12
- Saem da v1: a gestão de usuários pelo painel e a tela `/conta/senha`. A tela `/plugins/n3x/core/usuarios` vira uma lista só leitura para o root, com nome e estado.
- Qualquer mudança é feita editando o yaml e reiniciando o controller. O `env.Sessions().Revoke` não é usado na v1.

## Justificativa
- O yaml é durável e versionado no repositório de deploy, o que dá rastreabilidade das mudanças mesmo sem auditoria.
- Elimina telas e fluxos de escrita na v1.

## Trade-off
- Toda mudança de usuário exige editar o yaml e reiniciar o controller.
- O usuário não troca a própria senha. Quem gera o hash é o operador.

## Requisitos relacionados
FR-008, FR-009.
