# WidgetHub — Product Requirements (v1)

## Vision
A web app that lists useful online widgets as tiles on a homepage. Each widget opens on its own page. No login in v1 (planned for a later version).

## v1 scope
| # | Requirement | Notes |
|---|---|---|
| R1 | React SPA, opens directly with no login | Vite + React + TypeScript |
| R2 | Homepage = grid of widget tiles | Each tile: icon (in-repo SVG, no icon-font/library dependency — see Clarification C5), name, one-line description, links to widget route |
| R3 | Widget 1: Calculator | Basic arithmetic (+ - × ÷), decimals, %, sign toggle, clear/backspace, keyboard input |
| R4 | Widget 2: Scientific calculator | Trig (deg/rad), inverse trig, log/ln, powers/roots, factorial, constants (π, e), parentheses, memory, history — memory and history are session-only state, not persisted (see Clarification C6) |
| R5 | Widget 3: Unit converter | Categories: length, mass, temperature, volume, area, speed, time, data. Two-way live conversion, swap button |
| R6 | Widgets are registered in one registry | Adding widget #4 = new folder + one registry entry, no changes to Home |
| R7 | Responsive and accessible | Works 360px to desktop; keyboard navigable; WCAG 2.1 AA contrast (4.5:1 normal text, 3:1 large text/UI components); labelled controls; supported browsers per Clarification C3 |
| R8 | Tested | Unit tests on all logic engines, component tests per widget, Playwright smoke test per route |

## Out of scope for v1
Login/accounts, backend/API, saved user data (including persisted calculator memory/history — see C6), analytics, i18n, deployment pipeline (decide later — see Proposed answers, item 4), unmatched-route handling beyond a static 404 page (see C4).

## Clarifications
These resolve PRD ambiguities identified during T0.1 review with a concrete default, so the architect (T0.2) is not blocked. They are not open questions — no human sign-off required — but can be revisited if a human disagrees.

- **C1 — Accessibility metric (R7 vs AC6).** A Lighthouse accessibility score is an automated smoke check, not proof of WCAG 2.1 AA conformance by itself. Both are required for "done": Lighthouse accessibility score >= 90 on every route (AC6) **and** AA contrast/keyboard/label checks (manual pass and/or axe-core in CI, per R7 and T5.1) — contrast >= 4.5:1 for normal text and >= 3:1 for large text (>=24px, or >=19px/14pt bold) and for UI component boundaries/graphical objects. Source: WCAG 2.1 Success Criteria 1.4.3 and 1.4.11 (W3C).

- **C2 — AC5 tolerance and reference constants.** Internal (pre-rounding, double-precision) conversion results must match the reference values within relative error <= 1e-9. Reference values use exact, internationally defined constants, not rounded approximations:
  - 1 in = 2.54 cm (exact) — International Yard and Pound Agreement, 1959, as codified in NIST SP 811 (Guide for the Use of the International System of Units), Appendix B.
  - 1 mile = 1609.344 m (exact) — derived from the inch definition above.
  - 1 lb (avoirdupois) = 0.45359237 kg (exact) — International Yard and Pound Agreement, 1959.
  - °F = °C × 9⁄5 + 32 (exact, defined relationship, not a measured constant) — NIST SP 811, §4.
  - 1 US gallon = 3.785411784 L (exact) — NIST Handbook 44, Appendix C.
  Displayed (rounded) values follow C8; AC5's tolerance applies to the underlying engine output, not the rounded UI string.

- **C3 — Browser support matrix.** Current stable release + previous major release of Chrome, Firefox, Safari, and Edge, on both desktop and mobile (iOS Safari, Chrome for Android). No IE11 or legacy (EdgeHTML) Edge. Playwright smoke tests (R8) should cover at minimum Chromium, Firefox, and WebKit engines to approximate this matrix.

- **C4 — Unmatched routes (404).** Any URL that doesn't match a registered route (Home or a widget route) renders a dedicated 404 page — not a silent redirect to Home — with a heading, a short message, and a link back to Home. Implemented as a catch-all (`*`) route (ties to T1.1).

- **C5 — Tile icon source.** Icons are custom, in-repo SVGs, one per widget (~24-32px, `currentColor` fill so they inherit theme/contrast), authored as React components in a shared location (e.g. `src/icons/`). No external icon-font or icon-library dependency, consistent with CLAUDE.md's "no new dependency without architect approval." Icons are decorative (`aria-hidden="true"`); the tile link's accessible name comes from the widget name/description text, not the icon.

- **C6 — History/memory persistence (R4) vs. "no saved user data."** Scientific-calculator memory (M+/M-/MR/MC) and history are in-memory application/component state only. They reset on full page reload/navigation away and are never written to `localStorage`, cookies, `sessionStorage`, or a backend. This keeps R4's "history" and "memory" scoped to *within the current session* and does not contradict the "no saved user data" out-of-scope item.

- **C7 — Number display limits.** Inputs and results display up to 15 significant digits, matching the range of digits an IEEE-754 double can round-trip reliably (see MDN `Number` precision notes / ECMA-262 `Number` section). A non-zero result with magnitude >= 1e15 or magnitude < 1e-9 switches to exponential notation (e.g. `1.23e+21`). A result outside representable range (overflow) displays a clear `Error` / `Overflow` state — never the literal strings `Infinity` or `NaN` (this generalizes AC4's divide-by-zero rule to all overflow cases). Widget-specific formatting detail (grouping separators, max decimal places, etc.) is defined in each widget's own spec (T2.1 / T3.1 / T4.1) and must not contradict this default.

- **C8 — Display rounding for conversions.** Converted values in the Unit Converter UI are rounded to 6 significant figures by default for readability; the value backing the "other" input field keeps full double precision internally so repeated swaps don't accumulate rounding error. Category-specific rounding (e.g. whole-number-friendly units) is defined in T4.1. AC5's tolerance (C2) is checked against the unrounded engine value.

## Acceptance criteria (summary)
- AC1: `npm run build`, `npm run lint`, `npm run typecheck`, `npm test` all pass with zero errors.
- AC2: Homepage shows 3 tiles; each navigates to /calculator, /scientific-calculator, /unit-converter and back. An unmatched route renders the 404 page (C4), not a blank page or crash.
- AC3: Calculator/scientific engines never use `eval` / `new Function`. Expression parsing is a tested pure module.
- AC4: 0.1 + 0.2 displays 0.3 (float-error handling); divide by zero shows a clear error state, not `Infinity`/`NaN` (see C7 for the general overflow rule).
- AC5: Unit conversions match the reference values in C2 (e.g. 0 °C = 32 °F, 1 mi = 1.609344 km, 1 lb = 0.45359237 kg) within relative error <= 1e-9 on the unrounded engine output.
- AC6: Lighthouse accessibility score >= 90 on each route, **and** the manual/axe-core AA checks in C1 pass (contrast, keyboard nav, labels) — Lighthouse alone is not sufficient.
- AC7: All routes render usably and pass the Playwright smoke test (R8) on the browser matrix in C3.

## Open questions
- Deployment target (Vercel, Netlify, GitHub Pages, other)? — deferred by human on 2026-09-30; revisit before release.

## Answers to open questions
Items 1-3 **DECIDED** by the human on 2026-09-30 (recorded in docs/DECISIONS.md). Item 4 remains **OPEN — deferred**.

1. **Branding/name** — DECIDED: keep "WidgetHub" as the v1 product name. *Rationale:* already used throughout the repo (CLAUDE.md title, docs, likely package name); renaming later only touches copy/metadata, so it doesn't need to block architecture or scaffolding.

2. **Colour palette** — DECIDED: Tailwind's default neutral/slate scale for surfaces and text, plus one accessible accent colour (e.g. Tailwind `indigo-600`/`blue-600`) for interactive elements, chosen/verified to meet the 4.5:1 contrast bar in C1 on a white/near-white background. *Rationale:* adds no new dependency (Tailwind is already chosen), the default scale is well-tested for accessible contrast, and it unblocks T0.3 without a separate design pass.

3. **Dark mode** — DECIDED: out of scope for v1, targeted for v1.1. *Rationale:* not part of R1-R8; supporting it now would double the AA-contrast surface (C1) to verify across every widget before ship, for a feature not in the current R-list. Tailwind's `dark:` variant makes it a low-cost additive follow-up once light mode ships.

4. **Deployment target** — OPEN (deferred by human; candidate, not decided): Vercel, with Netlify as an equivalent fallback. *Rationale:* both are zero-config for Vite SPAs, have a free tier sufficient for v1, and handle the client-side routing rewrite that React Router's widget routes (e.g. `/calculator`) require out of the box. Deferred to whichever platform the human already has an account/org on; this doesn't block T0.1-T4.x, only T0.5 (CI) and release.
