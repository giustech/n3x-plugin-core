# ADR-005: Token de sessão com nonce, TTL de 1 h e identidade amarrada

Status: Aceito
Data: 2026-10-20

## Contexto
O TTL da sessão estava pendente no HLD de plataforma e no ADR-003 do SDK, com a hipótese de 1 h. Também faltava garantir que um sidecar não se apresentasse com a identidade de outro serviço, e havia a necessidade de distinguir duas sessões da mesma identidade para detectar conflito.

## Decisão
- O token é HMAC-SHA256 sobre serviço, instância, emitido em, expira em e um nonce aleatório.
- TTL de 1 h (configurável em `session.ttl`), com renovação em 80% do TTL e jitter de 10%.
- A validação usa `hmac.Equal`, em tempo constante.
- `Register`, `Heartbeat`, `Event` e `Exit` só aceitam a identidade igual à do token. Se forem diferentes, a resposta é `PERMISSION_DENIED`, com um log de alerta.
- O nonce identifica a sessão: dois nonces ativos para a mesma identidade caracterizam conflito.

## Justificativa
- TTL curto, com renovação silenciosa, limita a janela de abuso de um token vazado.
- Amarrar a identidade impede que a credencial de um serviço seja usada para envenenar o inventário de outro.

## Trade-off
- Uma renovação a cada 48 min por sidecar gera uma carga pequena no controller.
- O token não é revogável individualmente na v1. A revogação geral é a rotação da chave.

## Requisitos relacionados
FR-002, FR-005, FR-006, FR-016; segurança.
