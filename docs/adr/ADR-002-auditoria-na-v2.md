# ADR-002: Auditoria adiada para a v2

Status: Aceito
Data: 2026-10-20

## Contexto
O FR-010 pede uma trilha de auditoria de 90 dias, só com inclusões, gravada pelo core. Com o armazenamento em memória (ADR-001), essa trilha seria apagada a cada restart ou deploy do controller, o que não cumpre o propósito de auditoria.

## Decisão
- O FR-010 sai da v1 inteiro.
- O core não fornece o gravador de auditoria. A capacidade `WriteAudit` continua reservada no SDK para a v2.
- O login do painel no controller deixa de depender da gravação de um evento de auditoria. Com isso saem o `audit_fallback=true`, a recusa de ações por falha de auditoria, a métrica `n3x_controller_audit_failures_total`, a tela `/plugins/n3x/core/auditoria` e o `audit_retention`.
- Como paliativo, o login, o logout e o bloqueio do painel continuam no log JSON operacional do host do controller.

## Justificativa
- Uma trilha que some no restart passaria uma falsa sensação de conformidade.
- A auditoria volta junto com o banco da v2, atrás da mesma porta `Store`.

## Trade-off
- Não há trilha formal de acesso ao painel na v1. O log operacional não garante inclusão só, nem retenção.
- O PRD precisa registrar a lacuna de compliance.

## Requisitos relacionados
FR-008, FR-009, FR-010; compliance e LGPD.
