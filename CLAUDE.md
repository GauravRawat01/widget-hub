# WidgetHub

Web app listing online widgets as tiles. v1 = Calculator, Scientific Calculator, Unit Converter. No login yet.
Source of truth: `docs/PRD.md` (what) and `docs/TASKS.md` (work plan). Architecture: `docs/ARCHITECTURE.md` (once written).

## Stack
Vite + React + TypeScript (strict), Tailwind CSS, React Router, Vitest + React Testing Library, Playwright.

## Commands (available after T0.3/T0.4)
- `npm run dev` — dev server
- `npm run lint` / `npm run typecheck` / `npm test` / `npm run test:e2e` / `npm run build`
- Definition of done: lint, typecheck, test and build all pass.

## Conventions
- One folder per widget: `src/widgets/<name>/` with `index.tsx`, `logic/` (pure TS, no React), `__tests__/`.
- Widgets register in `src/widgets/registry.ts`. Home page is generated from the registry; never hard-code tiles.
- Logic lives in pure functions/modules; UI components stay thin.
- Never use `eval` or `new Function`. No `any` without a comment explaining why.
- Accessibility is required: labelled controls, keyboard support, visible focus, AA contrast.
- Small, focused commits, one task ID per branch: `feat/T2.2-calc-engine`. Commit format: `T2.2: add calculator engine`.

## Agent workflow
Run the main session as the orchestrator: `claude --agent orchestrator`.
Flow per task: requirements-analyst -> architect (if design needed) -> coder -> tester -> code-reviewer -> docs-writer.
Agents communicate through files (`docs/`, task status in `docs/TASKS.md`), not long chat summaries.
Only the orchestrator edits `docs/TASKS.md` statuses. Reviewers never edit code; they report findings.

## Rules for all agents
- Read this file and the relevant task in `docs/TASKS.md` before starting.
- Stay inside your role; hand back with a short summary: what changed, files touched, open issues.
- Do not add dependencies without the architect's approval (record the reason in `docs/DECISIONS.md`).
- Never commit secrets. Never push to main directly.
