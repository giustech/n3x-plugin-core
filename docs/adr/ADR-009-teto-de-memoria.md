# ADR-009: Teto de memória, ordem de descarte e estado degradado

Status: Aceito
Data: 2026-10-20

## Contexto
Com tudo em memória (ADR-001), o core precisa de um limite explícito para não estourar o orçamento do controller (150 MB e 0,15 vCPU com 500 sidecars). O estado `degradado`, antes associado ao "banco fora do ar", precisa de um novo significado.

## Decisão
- **Teto:** `memory_limit` na seção do core, com padrão de 128 MiB, medido por contabilidade aproximada do tamanho de configs, manifestos, eventos e instâncias.
- **Retenções:**
  - histórico de 24 h (`inventory_retention`)
  - até 20 versões de config por serviço
  - até 200 eventos por instância
- **Descarte:** acima de 90% do teto, o core descarta nesta ordem:
  1. o histórico
  2. os eventos mais antigos
  3. as versões de config sem instância apontando
- **Degradado:** se mesmo assim o teto estourar:
  - o core se marca `degradado`, com banner no painel e `n3x_n3x_core_degraded = 1`
  - continua aceitando heartbeats e mudanças de estado
  - recusa novas versões de config com `RESOURCE_EXHAUSTED`
  - sai do estado degradado sozinho quando o uso cai abaixo de 90%
- **Snapshots imutáveis por serviço** atendem a leitura do painel sem lock.

## Justificativa
- O descarte é explícito e em ordem de menor valor, ao contrário do descarte silencioso da memória do SDK.
- Preserva o estado vivo (quem está conectado), que é o mais importante para o operador.

## Trade-off
- A contabilidade é aproximada, então o uso real pode passar um pouco do teto.
- Configs novas podem ficar sem registro enquanto o core estiver degradado.

## Requisitos relacionados
FR-006, FR-007, FR-008, FR-016; recursos do controller.
