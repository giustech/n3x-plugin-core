# ADR-008: Core sem hooks de requisição

Status: Aceito
Data: 2026-10-20

## Contexto
O orçamento de latência do sidecar (p50 ≤ 0,5 ms e p99 ≤ 2 ms com o core ativo) e a regra de que o controller só é chamado em segundo plano (ADR-016 do SDK) limitam o que o core pode fazer no caminho da requisição. Nada do que o core faz depende de uma requisição específica.

## Decisão
- A metade de proxy do core não declara nenhuma fase nem hook de requisição.
- Login, registro, heartbeat, eventos e saída rodam em tarefas `env.Background()`, dentro do limite de 8 tarefas por plugin.
- O manifesto do core declara as duas metades, `controller.required: true`, e nenhuma fase.

## Justificativa
- O custo do core no caminho da requisição é zero, e uma falha dele nunca afeta o tráfego da aplicação.
- Serve de referência clara de plugin de controle para outros autores.

## Trade-off
- O core não consegue observar o tráfego. Isso é aceito, porque observabilidade do tráfego está fora da v1.

## Requisitos relacionados
FR-004, FR-012, FR-016; latência e alocações.
