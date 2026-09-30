# WidgetHub — v1 Task Backlog

Legend: **Owner** = the agent that does the work. Every task ends with tester + code-reviewer sign-off unless marked otherwise.
Status values: `todo` | `in-progress` | `blocked` | `done`. Update this file as tasks move (orchestrator owns edits).

## Phase 0 — Foundation
| ID | Task | Owner | Depends on | Status |
|---|---|---|---|---|
| T0.1 | Confirm PRD, resolve open questions | requirements-analyst | — | done |
| T0.2 | Architecture note: folder layout, widget registry contract, routing, state approach (docs/ARCHITECTURE.md) | architect | T0.1 | todo |
| T0.3 | Scaffold Vite + React + TS, ESLint, Prettier, Tailwind, React Router | coder | T0.2 | todo |
| T0.4 | Test harness: Vitest + RTL + Playwright, npm scripts, coverage threshold | tester | T0.3 | todo |
| T0.5 | CI workflow (lint, typecheck, test, build) on push/PR | coder | T0.4 | todo |
| T0.6 | Git hygiene: .gitignore, branch + PR conventions in CONTRIBUTING notes | docs-writer | T0.3 | todo |

## Phase 1 — Shell and Home
| ID | Task | Owner | Depends on | Status |
|---|---|---|---|---|
| T1.1 | App shell: layout, header, routes, 404 page | coder | T0.3 | todo |
| T1.2 | Widget registry (`src/widgets/registry.ts`) with typed `WidgetDefinition` | coder | T0.2 | todo |
| T1.3 | Home page with tile grid generated from registry | coder | T1.1, T1.2 | todo |
| T1.4 | Home + routing tests (unit + e2e smoke) | tester | T1.3 | todo |

## Phase 2 — Widget 1: Calculator
| ID | Task | Owner | Depends on | Status |
|---|---|---|---|---|
| T2.1 | Spec: behaviours, edge cases, keyboard map | requirements-analyst | T1.2 | todo |
| T2.2 | Pure calculation module (state machine, decimal-safe arithmetic) | coder | T2.1 | todo |
| T2.3 | Unit tests for engine (edge cases: chained ops, %, divide by zero, precision) | tester | T2.2 | todo |
| T2.4 | Calculator UI component + keyboard support | coder | T2.2 | todo |
| T2.5 | Component + e2e tests | tester | T2.4 | todo |
| T2.6 | Register widget, review, merge | code-reviewer | T2.5 | todo |

## Phase 3 — Widget 2: Scientific Calculator
| ID | Task | Owner | Depends on | Status |
|---|---|---|---|---|
| T3.1 | Spec incl. operator precedence, deg/rad, error states | requirements-analyst | T2.6 | todo |
| T3.2 | Tokenizer + parser (shunting-yard or recursive descent), no eval | coder (design reviewed by architect) | T3.1 | todo |
| T3.3 | Parser/evaluator tests (precedence, nesting, functions, invalid input, large numbers) | tester | T3.2 | todo |
| T3.4 | Scientific UI: function keypad, mode toggles, memory, history | coder | T3.2 | todo |
| T3.5 | Component + e2e tests | tester | T3.4 | todo |
| T3.6 | Register widget, review, merge (reuse Calculator display/keypad components) | code-reviewer | T3.5 | todo |

## Phase 4 — Widget 3: Unit Converter
| ID | Task | Owner | Depends on | Status |
|---|---|---|---|---|
| T4.1 | Spec: categories, units, precision/rounding rules | requirements-analyst | T1.2 | todo |
| T4.2 | Conversion data model (base-unit factors; temperature as special case) | coder | T4.1 | todo |
| T4.3 | Tests against reference values, round-trip property tests | tester | T4.2 | todo |
| T4.4 | Converter UI: category tabs, two-way inputs, swap | coder | T4.2 | todo |
| T4.5 | Component + e2e tests | tester | T4.4 | todo |
| T4.6 | Register widget, review, merge | code-reviewer | T4.5 | todo |

## Phase 5 — Polish and release
| ID | Task | Owner | Depends on | Status |
|---|---|---|---|---|
| T5.1 | Responsive + accessibility pass (axe, keyboard, screen-reader labels) | tester + coder | T2.6, T3.6, T4.6 | todo |
| T5.2 | Performance: route-level code splitting, bundle size check | coder | T5.1 | todo |
| T5.3 | README (run, test, add-a-widget guide), CHANGELOG | docs-writer | T5.1 | todo |
| T5.4 | Full-repo review against PRD acceptance criteria | code-reviewer | T5.2 | todo |
| T5.5 | Tag v0.1.0 | orchestrator | T5.4 | todo |

## Parallelism notes
After Phase 1, widgets 1-3 are independent: three worktrees (one per widget) can run in parallel. Only T5.x needs all three finished.
