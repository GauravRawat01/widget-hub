# Decision log (ADR-lite)
Format: `YYYY-MM-DD — Decision — Why — Alternatives considered`

- 2026-09-30 — Vite + React + TypeScript — fast, minimal config, type safety for logic-heavy widgets — Next.js (overkill for v1), plain JS.
- 2026-09-30 — Vitest + RTL + Playwright — same toolchain as Vite, unit/component/e2e coverage — Jest.
- 2026-09-30 — Tailwind CSS + React Router (route per widget) — fast UI work, shareable URLs — CSS modules.
- 2026-09-30 — Auth, backend, deployment deferred beyond v1.
- 2026-09-30 — T0.1: product name stays "WidgetHub" — already used everywhere; renaming later is cheap — new name now.
- 2026-09-30 — T0.1: palette = Tailwind default slate/neutral + one AA-verified accent (e.g. indigo-600) — no new dependency, proven contrast — custom design system.
- 2026-09-30 — T0.1: dark mode deferred to v1.1 — not in R1-R8; would double the AA contrast verification surface — ship in v1.
- 2026-09-30 — T0.1: deployment target left open (human deferred) — not needed until release — Vercel/Netlify/GitHub Pages candidates.
- 2026-09-30 — T0.1: PRD clarifications C1-C8 accepted as defaults (a11y metric, 1e-9 conversion tolerance, browser matrix, 404 page, in-repo SVG icons, session-only memory/history, 15-sig-digit display, 6-sig-fig converter display) — unblock architecture — leave ambiguous.
- 2026-09-30 — Default branch renamed master -> main — matches CLAUDE.md conventions.
- 2026-09-30 — Branching: `dev` is the integration branch; task branches (`feat/<TaskID>-<slug>`) are cut from `dev` and merged back via PR; `dev` -> `main` handled separately by the human (maybe automated monthly later) — keeps `main` stable while tasks integrate continuously — trunk-based on `main`.
