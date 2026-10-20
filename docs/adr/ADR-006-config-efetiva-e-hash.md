# ADR-006: Config efetiva em JSON canônico, impressão digital de segredos e hash SHA-256

Status: Aceito
Data: 2026-10-20

## Contexto
Os HLDs do sidecar e do SDK deixaram para este HLD o formato da config efetiva, o mascaramento e o hash. O hash precisa corresponder sempre à config em uso, nenhum segredo pode sair do sidecar, e a troca de um segredo deve aparecer como mudança.

## Decisão
- **Formato 1:** JSON canônico pela RFC 8785, com a estrutura `{format, sidecar, plugins: [{id, instance, phase, order, failure, config}]}`.
- **Mascaramento:** todo campo marcado como segredo no schema (`plugin.Secret`), e também o `controller.secret` do sidecar, vira `***:` seguido dos 12 primeiros caracteres hex do SHA-256 do texto `ENC[...]`. A impressão digital é calculada sobre o cifrado, nunca sobre o valor em claro.
- **Segredo sem `ENC[...]`:** vira `***:plain`, e o sidecar emite o evento `secret_in_clear`. O valor não sai do processo.
- **Hash:** `sha256:<hex>` do JSON canônico mascarado. O controller recalcula o hash e, se não bater, recusa com `DATA_LOSS`. Nesse caso, o painel mantém a última versão válida e marca um alerta.
- **Armazenamento:** as versões são deduplicadas pelo hash, com até 20 por serviço.

## Justificativa
- A forma canônica garante o mesmo hash para a mesma config, independentemente da ordem das chaves.
- A impressão digital sobre o cifrado revela a troca de um segredo sem permitir ataque de dicionário ao valor.
- O recálculo no controller detecta corrupção sem precisar de um campo extra.

## Trade-off
- O core precisa de uma implementação própria da RFC 8785.
- A impressão digital muda também quando o mesmo segredo é cifrado de novo (outro nonce do AES-GCM), o que gera uma falsa mudança.

## Requisitos relacionados
FR-006, FR-007, FR-016; confiabilidade e segurança.
