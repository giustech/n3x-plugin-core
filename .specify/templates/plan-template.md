# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]

**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: [e.g., Python 3.11, Swift 5.9, Rust 1.75 or NEEDS CLARIFICATION]

**Primary Dependencies**: [e.g., FastAPI, UIKit, LLVM or NEEDS CLARIFICATION]

**Storage**: [if applicable, e.g., PostgreSQL, CoreData, files or N/A]

**Testing**: [e.g., pytest, XCTest, cargo test or NEEDS CLARIFICATION]

**Target Platform**: [e.g., Linux server, iOS 15+, WASM or NEEDS CLARIFICATION]

**Project Type**: [e.g., library/cli/web-service/mobile-app/compiler/desktop-app or NEEDS CLARIFICATION]

**Performance Goals**: [domain-specific, e.g., 1000 req/s, 10k lines/sec, 60 fps or NEEDS CLARIFICATION]

**Constraints**: [domain-specific, e.g., <200ms p95, <100MB memory, offline-capable or NEEDS CLARIFICATION]

**Scale/Scope**: [domain-specific, e.g., 10k users, 1M LOC, 50 screens or NEEDS CLARIFICATION]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Portões da constituição (`.specify/memory/constitution.md`, v1.0.0). Marque cada item; toda violação
precisa de justificativa em "Complexity Tracking".

- [ ] **I. Fora do caminho da requisição:** nenhuma fase ou hook de requisição; trabalho só em
  `env.Background()`; falha do core ou do controller não afeta o tráfego.
- [ ] **II. Segredo nunca sai:** nenhum segredo, token ou config sem máscara em log, erro, métrica, evento
  ou tela; teste de canário (`secrettest.Leaks` = 0) previsto.
- [ ] **III. Sidecar como fonte de verdade:** nada escreve config de sidecar; estado reconstruível depois
  de restart; identidade vem do token.
- [ ] **IV. Contratos só por adição:** mudanças de proto, formato da config efetiva e schema são aditivas;
  `buf breaking` e goldens cobertos.
- [ ] **V. Memória limitada:** todo dado novo tem teto, retenção e lugar na ordem de descarte; estado
  degradado tratado.
- [ ] **VI. Dono único e mínimo privilégio:** papéis e rotas corretos; nenhuma rota pública nova; regras
  de LGPD respeitadas.
- [ ] **VII. Plugin de referência:** só APIs públicas do SDK; análise estática sem `//n3x:allow`; regras
  de import; métricas só no catálogo `internal/metricdefs`.
- [ ] **VIII. Teste primeiro:** testes escritos antes e falhando; cobertura ≥ 90%; `-race`, fuzzing e
  benchmarks que medem as metas numéricas.
- [ ] **IX. Simplicidade na v1:** sem banco, disco, auditoria, tracing ou réplica extra; armazenamento só
  pela porta `Store`.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.

  Neste repositório (n3x-plugin-core), use a estrutura de plugin do SDK na raiz
  do módulo Go: `proto/`, `types/`, `effconfig/`, `agent/`, `controller/`,
  `internal/` e `tools/`, com testes `_test.go` no mesmo pacote do código.
-->

```text
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
