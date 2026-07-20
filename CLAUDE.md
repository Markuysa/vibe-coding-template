# {{PROJECT_NAME}}

## Stack
{{STACK}}  <!-- e.g. Next.js 15, TypeScript strict, Tailwind, Supabase -->

## Commands
- dev: {{DEV_CMD}}
- test: {{TEST_CMD}}
- lint: {{LINT_CMD}}
- build: {{BUILD_CMD}}

## Rules
- Non-trivial changes go through a plan first: propose, then write code.
- Tests are mandatory for business logic.
- Design tokens come only from {{TOKENS_FILE}}.
- Never touch {{PROTECTED_PATHS}} without explicit permission.

## References (read on demand, do not hold in context)
- Spec: docs/PRD.md
- Architecture: docs/ARCHITECTURE.md
- Cheat sheet: docs/QUICK_REF.md
- Decisions (ADR): docs/decisions/

## Compaction policy
When compacting, preserve: the full list of changed files, test commands, and decisions
with their reasoning. Condense research findings aggressively.

<!-- Keep this file under 200 lines. Procedures belong in .claude/skills/, where they
     cost nothing until invoked; this file is loaded into context every session. -->
