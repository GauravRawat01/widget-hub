# WidgetHub — Architecture (v1)

Status: T0.2, 2026-09-30 (rev. 4 after review). Owner: architect. Inputs: `docs/PRD.md` (R1-R8, C1-C8, AC1-AC7), `docs/DECISIONS.md`.
Changes to anything in sections 2-5 need an architect review and a `docs/DECISIONS.md` entry.

## 1. Repository layout

```
widget-hub/
  index.html                  # Vite entry, <html lang="en">, <div id="root">
  package.json                # scripts in section 7; "engines": { "node": ">=22" }
  .nvmrc                      # 24 (current Node LTS)
  vite.config.ts              # Vite + Vitest config (single file), '@' alias
  tsconfig.json               # references only: tsconfig.app.json, tsconfig.node.json
  tsconfig.app.json           # src/** (DOM libs, jsx)
  tsconfig.node.json          # vite.config.ts, playwright.config.ts, e2e/**/*.ts (types: node)
  eslint.config.js            # flat config (section 7); JS, not in any tsconfig
  .prettierrc.json  .prettierignore
  playwright.config.ts
  public/favicon.svg
  e2e/                        # Playwright specs: smoke.spec.ts, <widget-id>.spec.ts
  docs/
  src/
    main.tsx                  # createRoot + <App/>; imports index.css
    index.css                 # @import "tailwindcss"; @theme tokens (section 7)
    app/                      # shell: App.tsx, router.tsx, AppLayout.tsx, WidgetRoute.tsx,
                              #        RouteErrorPage.tsx, __tests__/
    pages/                    # HomePage.tsx, NotFoundPage.tsx, __tests__/
    components/               # shared presentational UI: WidgetTile, Button, keypad/ (Display,
                              #   Keypad, Key — reused by both calculators, T3.6), __tests__/
    hooks/                    # shared React hooks: useDocumentTitle, useKeyboardShortcuts
    lib/                      # shared PURE TS (no React): number/ (section 5), __tests__/
    test/setup.ts             # Vitest setup: jest-dom matchers, RTL cleanup
    widgets/
      types.ts                # WidgetDefinition (section 2)
      registry.ts             # the only list of widgets
      __tests__/registry.test.ts  # imports ../registry; excluded from widget import overrides (section 7)
      calculator/
        icon.tsx              # C5: SVG component; imports nothing from index.tsx or logic/
        index.tsx             # default export: widget page component (thin)
        components/           # widget-private UI (optional)
        logic/                # pure TS: engine.ts (reducer), keymap.ts
        __tests__/
      scientific-calculator/
        icon.tsx  index.tsx
        logic/                # tokenizer.ts, parser.ts (recursive descent -> AST), evaluate.ts,
                              #   engine.ts (reducer incl. memory/history), keymap.ts
        __tests__/
      unit-converter/
        icon.tsx  index.tsx
        logic/                # units.ts (data), convert.ts
        __tests__/
```

Import rules (enforced by ESLint, section 7):
- `src/lib/**` may import only other `src/lib` files (same folder via `./`, anything else via `@/lib/...`). `src/widgets/*/logic/**` may import only `@/lib/...` and files within the same `logic/` folder (`./`). Neither may import `react*`, touch `window`/`document`/`globalThis`, or use storage.
- A widget may import from `@/components`, `@/hooks`, `@/lib`, and its own folder only. Never from another widget folder. Code shared by two widgets moves to `components/` or `lib/`. Widget subfolders are one level deep.
- `icon.tsx` imports nothing local (keeps it out of the widget's lazy chunk dependency graph).
- Folder name = widget `id` (kebab-case). Shared UI icons (if any are ever needed) go in `src/components/icons/`.

## 2. Widget contract

```ts
// src/widgets/types.ts
import type { ComponentType, SVGProps } from 'react';

export type WidgetCategory = 'calculators' | 'converters';

export interface WidgetDefinition {
  /** Stable kebab-case identity; equals the folder name. Never renamed or reused
   *  (future: favourites/usage keyed by id once auth exists). */
  readonly id: string;
  /** URL path. Normally `/${id}`; kept separate so a URL can change without changing identity. */
  readonly path: `/${string}`;
  /** Tile title, page <h1>, document title. */
  readonly name: string;
  /** One line, <= 80 chars, shown on the tile. */
  readonly description: string;
  /** Not rendered in v1 (flat grid). Reserved for grouping/filtering; present now to avoid migration. */
  readonly category: WidgetCategory;
  /** Decorative in-repo SVG (C5) from `<id>/icon.tsx`; rendered aria-hidden="true", fill="currentColor". */
  readonly Icon: ComponentType<SVGProps<SVGSVGElement>>;
  /** Dynamic import of the widget folder; its default export is the page component (no props). */
  readonly load: () => Promise<{ default: ComponentType }>;
}
```

```ts
// src/widgets/registry.ts
import { CalculatorIcon } from './calculator/icon';
import type { WidgetDefinition } from './types';

export const widgets: readonly WidgetDefinition[] = [
  {
    id: 'calculator',
    path: '/calculator',
    name: 'Calculator',
    description: 'Basic arithmetic with keyboard support.',
    category: 'calculators',
    Icon: CalculatorIcon,
    load: () => import('./calculator'),
  },
  // scientific-calculator, unit-converter ... (array order = tile order on Home)
];
```

Rules:
- R6: adding widget #4 = a new folder `src/widgets/<id>/` + one registry entry. Home and router map over `widgets`; neither names a widget.
- Icons are imported statically (small, needed on Home); `index.tsx` is only reached through `load`, so each widget stays in its own chunk.
- `load` (not a pre-built `React.lazy`) keeps the registry plain data, testable, and preloadable on tile hover/focus later.
- `registry.test.ts` enforces: unique `id` and `path`; `path` matches `^/[a-z0-9-]+$` and is not `/`; description non-empty and <= 80 chars; every `load()` resolves to a module whose `default` is a React component type (`typeof d === 'function'`, or a non-null object with `$$typeof` for `memo`/`forwardRef`).
- The widget page `<h1>` is rendered by the shell (`WidgetRoute`), not by the widget.

## 3. Routing, titles, focus

- React Router v7, library/data mode: `createBrowserRouter` + `<RouterProvider>` in `src/app/App.tsx`. Clean URLs.
- `src/app/router.tsx` exports `routes` (reused by tests) and builds the tree once, at module scope:

```
/                     AppLayout (header with Home link, <main><Outlet/></main>) errorElement=RouteErrorPage
  index               HomePage       <h1>WidgetHub</h1>, title "WidgetHub"
  ...widgets.map(d => { path: d.path, element: <WidgetRoute def={d} Component={lazy(d.load)} /> })
  *                   NotFoundPage   <h1>Page not found</h1>, title "Page not found — WidgetHub", link Home (C4)
```

- `lazy(d.load)` is called once per widget at module load (never in render): route-level code splitting now; T5.2 only verifies the bundle.
- `WidgetRoute` sets title `"<name> — WidgetHub"` (via `useDocumentTitle`), renders `<h1 tabIndex={-1}>{def.name}</h1>` immediately, then `<Suspense fallback={<p>Loading…</p>}><Component/></Suspense>`. Suspense wraps only the lazy component, so the heading never waits for the chunk.
- Every page renders exactly one `<h1 tabIndex={-1}>`; the header has no `<h1>`.
- Focus lives in `AppLayout`: a `useEffect` on `location.pathname` (skipped on first render via a ref) focuses `main h1`. Focus is managed in one place; pages only provide the heading. Sole exception: `RouteErrorPage` (below), which renders outside `AppLayout`.
- `RouteErrorPage` handles thrown errors and failed chunk loads (message + reload/Home links). Unmatched URLs hit `*`, not the error page. As the root `errorElement` it replaces `AppLayout` (no header, no `AppLayout` focus effect), so it renders its own `<h1 tabIndex={-1}>Something went wrong</h1>` inside its own `<main>`, sets the title `"Something went wrong — WidgetHub"`, and focuses its `<h1>` in a mount `useEffect`.
- Hosting (undecided, platform-neutral): the host must rewrite non-file paths to `/index.html` with 200 (Vercel/Netlify rule). GitHub Pages would need a copied `404.html` and Vite `base`. `vite preview` does SPA fallback, so e2e covers deep links.

## 4. State

- No global store in v1. No Context except React Router's. Nothing is shared across widgets.
- Each widget engine is a pure reducer in `logic/engine.ts`:
  ```ts
  export interface CalculatorState { /* readonly fields */ }
  export type CalculatorAction = { type: 'digit'; digit: string } | { type: 'operator'; op: Operator } | ...;
  export const initialState: CalculatorState;
  export function reduce(state: CalculatorState, action: CalculatorAction): CalculatorState;
  ```
  UI: `const [state, dispatch] = useReducer(reduce, initialState)`. Engines are unit-tested with no DOM.
- Input being edited is kept as the raw string (`""`, `"-"`, `"1."`, `"0.10"`) and the number is derived from it when needed. Calculator: `entry: string` for the current operand. Converter: `{ categoryId, fromUnitId, toUnitId, input: string, inputSide: 'from' | 'to' }`; the other side is derived (parse -> `convert`, full precision -> format, C8). Invalid/partial input derives an empty other side, not an error.
- Keyboard: `logic/keymap.ts` exports `keyToAction(key: string): Action | null` (pure). `useKeyboardShortcuts` attaches a `keydown` listener while mounted and ignores events targeting `input`/`textarea`/`select`.
- C6: scientific memory and history live only in reducer state; they reset on unmount/reload. Storage is banned by lint (section 7).
- Scientific parser (T3.2, detail reviewed then): tokenizer -> recursive-descent parser -> AST -> `evaluate(ast, { angleMode })`. No `eval`/`new Function` (AC3).

## 5. Numbers, precision, formatting (C7, C8, AC4)

Decision: no decimal library. Native doubles; add/subtract round relative to their operands; multiply/divide are unrounded; display rounds to 15 significant digits.

```ts
// src/lib/number/result.ts
export type NumericError = 'divide-by-zero' | 'overflow' | 'domain' | 'syntax';
export type NumericResult = { ok: true; value: number } | { ok: false; error: NumericError };
/** Infinity/-Infinity -> 'overflow'; NaN -> fallback (default 'domain'); -0 -> 0. */
export function checked(value: number, nanError?: NumericError): NumericResult;

// src/lib/number/arithmetic.ts
export function add(a: number, b: number): NumericResult;       // operand-relative rounding
export function subtract(a: number, b: number): NumericResult;  // operand-relative rounding
export function multiply(a: number, b: number): NumericResult;  // unrounded
export function divide(a: number, b: number): NumericResult;    // unrounded; b === 0 -> 'divide-by-zero'
export function percent(x: number): NumericResult;              // x / 100 (not x * 0.01)

// src/lib/number/format.ts
export const MAX_SIGNIFICANT_DIGITS = 15;          // C7
export const CONVERTER_SIGNIFICANT_DIGITS = 6;     // C8
export interface FormatOptions {
  significantDigits: number;
  expUpper?: number;   // default 1e15 (C7)
  expLower?: number;   // default 1e-9 (C7)
}
export function formatNumber(value: number, options: FormatOptions): string;
export function roundToSignificant(value: number, digits: number): number; // Number(v.toPrecision(d))
```

Add/subtract rounding (rounds `r` to decimal place `e - 14`, i.e. 15 significant digits of the larger operand; one scaled form for all magnitudes):
```ts
const r = a ± b;
if (a === 0 || b === 0 || !Number.isFinite(r)) return checked(r);
const m = Math.max(Math.abs(a), Math.abs(b));
const e = Number(m.toExponential().split('e')[1]); // exact decimal exponent; Math.floor(Math.log10(m)) can be off by one near powers of 10
if (e < -300) return checked(r);                    // 10 ** -e loses precision / overflows (10 ** 324 = Infinity -> NaN); subnormal range stays unrounded
const scaled = e >= 0 ? r / 10 ** e : r * 10 ** -e; // |scaled| < 20, so toFixed never switches to exponential
const out = Number(`${scaled.toFixed(14)}e${e}`);
return checked(Number.isFinite(out) ? out : r);     // rounding up past MAX_VALUE must not turn a finite r into 'overflow'
```
- The `Number.isFinite(out)` fallback: e.g. `subtract(1.7976931348623157e308, 1e292)` has a finite `r`, but its 15-digit rounding (`1.79769313486232e308`) exceeds `Number.MAX_VALUE` and parses to `Infinity`; the unrounded `r` is returned instead.
- Rebuilding with an `e<exp>` string (instead of `Number(scaled.toFixed(14)) / 10 ** -e`) makes the result the correctly rounded double of the rounded decimal. Parse-then-scale rounds twice (once when parsing the scaled decimal to a double, again when dividing/multiplying), even when the power of ten is exact (`10 ** k` is exact only for `0 <= k <= 22`), which can add an ulp of error (e.g. `0.7000000000000001`). The string form rounds once.
- The old `p >= 0` branch (`Math.round(r / 10**p) * 10**p`) is replaced by this form. Same rounding place; the only differences are exact binary ties (`toFixed` rounds half away from zero, `Math.round` half toward +Infinity) and the final rounding (string parse is correctly rounded; `* 10**p` adds a rounding and `10**p` is inexact for `p > 22`). The algorithm was verified by execution in Node during review (27 cases, including the T2.3 list below); the T2.3 tests pin it.
- Inputs are at most 15 significant digits, so this discards only float noise, never a digit within 15 significant digits of the larger operand. Accepted: `(123456789012345 + 0.4) - 123456789012345` -> `0`, matching a 15-digit calculator.
- Known limit, accepted by the human 2026-09-30 (no decimal library): the 15th significant digit may differ by 1 from correct decimal rounding when the discarded tail is within ~0.25 unit of half (`scaled` is a double, so `toFixed` rounds its binary value, not the exact decimal result). A double-representation limit; about 0.7% of random 15-digit pairs, verified by review.
- Multiply/divide stay unrounded so `1 / 3 * 3` = `1`. Accepted consequence: chains like `1/3 + 1 - 1` then `* 3` show `0.99999999999999`, like a 15-digit hardware calculator.

Required T2.3 tests (all must display as shown): `0.1 + 0.2` -> `0.3`; `0.1 + 0.2 - 0.3` -> `0`; `1 / 3 * 3` -> `1`; `1000000.1 - 1000000` -> `0.1`; `123456.7 - 123456` -> `0.7`; `9.99999999999999 - 9.99999999999998` -> `1e-14` (C7 fixed range ends at 1e-9); `0.1` added 100 times then `- 10` -> `0`; `(123456789012345 + 0.4) - 123456789012345` -> `0`; `50%` -> `0.5`. Semantics of `a + b%` are defined by the T2.1 spec, not here.
Tiny-operand tests in `src/lib/number/__tests__` assert the returned value with `Object.is`: `add(1e-300, 1e-300)` -> `2e-300`; `add(5e-324, 5e-324)` -> `1e-323`; `subtract(1.1e-100, 1e-100)` -> `1e-101`. The first and third also display as `2e-300` / `1e-101`; subnormal display is not specified (`1e-323` formats as `9.88131291682493e-324`, 15 digits of the exact binary value).

`formatNumber(value, opts)` (preconditions `Number.isFinite(value)` and `expUpper <= 1e21`; the function clamps `expUpper = Math.min(expUpper, 1e21)` because `String(v)` is plain decimal only below 1e21):
1. `v = roundToSignificant(value, significantDigits)`; if `v === 0` return `"0"` (never `"-0"`). Thresholds apply to `v`, so `999999999999999.9` -> `1e15` -> exponential.
2. `|v| >= expUpper` or `|v| < expLower`: exponential from `v.toExponential(d - 1)`, then strip trailing mantissa zeros and a trailing `.` (`"1.00000000000000e-14"` -> `"1e-14"`), e.g. `"1.23e+21"`, `"1.5e-10"`.
3. `|v| >= 1e-6`: `String(v)` (already fixed notation in this range, shortest form).
4. `[expLower, 1e-6)`: hand-built fixed notation from the `toExponential` mantissa digits and exponent of `|v|`, with `"-"` prepended when `v < 0` (`1.2e-7` -> `"0.00000012"`, `-1.2e-7` -> `"-0.00000012"`), because `toPrecision`/`String` switch to exponential below 1e-6.
Never returns `"Infinity"`/`"NaN"`.

Scientific (T3.1/T3.2 guidance):
- Degree mode: reduce the argument mod 360 before converting. `tan` returns `'domain'` when the exact degree value satisfies `((deg % 180) + 180) % 180 === 90` (checked before conversion, not by testing a float result). Trig results with `|r| < 1e-15` snap to 0 (`sin(180°)` = 0); this snap is degree mode only.
- Radian mode: no special case and no snap; `sin(1e-16)` returns `1e-16`; `tan(Math.PI / 2)` ≈ `1.633e16` is a normal result (displayed exponential).
- Factorial of a non-integer or negative number -> `'domain'`; `n > 170` -> `'overflow'`.
- Every engine returns `NumericResult`; widgets map `NumericError` to user text per their spec. The UI never renders `Infinity`/`NaN`.
- Converter: engine output is unrounded (C2 tolerance checked on it); UI uses `formatNumber(v, { significantDigits: 6 })`.

Not adopted (NEEDS HUMAN APPROVAL if ever wanted): decimal.js (~30 KB+; exact decimal basic ops, but a second numeric type across engines, trig/log still approximate, C7 formatting still needed). big.js lacks scientific functions. Revisit only if T2.3/T3.3 tests expose cases the approach above cannot meet.

## 6. Testing

| Layer | Tool | Location | Scope |
|---|---|---|---|
| Unit | Vitest | `src/lib/**/__tests__`, `src/widgets/*/__tests__/*.test.ts` | engines, parser, conversions, formatting, registry invariants |
| Component | Vitest + RTL + user-event (jsdom) | `src/**/__tests__/*.test.tsx` | Home tiles from registry, 404, titles/focus, each widget UI + keyboard |
| E2E | Playwright | `e2e/*.spec.ts` | smoke per route + key flows |

- `vite.config.ts` uses `defineConfig` from `vitest/config`. `resolve.alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) }`. `test: { environment: 'jsdom', setupFiles: ['src/test/setup.ts'], include: ['src/**/*.test.{ts,tsx}'] }`. Pure logic tests may add `// @vitest-environment node`.
- T0.4 ships one real test (App renders the Home `<h1>` via `createMemoryRouter(routes)`); `passWithNoTests` stays off.
- Router tests use `createMemoryRouter` with the exported `routes`.
- `e2e/smoke.spec.ts` discovers tiles from the DOM on `/`, visits each, checks `<h1>` text, focus on the `<h1>` after navigation, title and no console errors, then checks an unknown URL renders the 404. New widgets need no spec edits. Per-widget flows: `e2e/<id>.spec.ts`.
- Playwright: projects `chromium`, `firefox`, `webkit`, `mobile-chromium` (360x800 viewport, R7), `mobile-webkit` (`devices['iPhone 13']`, C3 iOS Safari). `webServer: { command: 'npm run build && npm run preview -- --port 4173 --strictPort', url: 'http://localhost:4173', reuseExistingServer: !process.env.CI }`; `use.baseURL` the same URL. Retries 2 on CI, 0 locally.
- Coverage (`@vitest/coverage-v8`), enforced by `npm test`: `include: ['src/**/*.{ts,tsx}']`; `exclude: ['src/main.tsx', 'src/test/**', 'src/widgets/types.ts', 'src/widgets/*/icon.tsx', '**/__tests__/**', '**/*.d.ts']`.
  - `src/lib/**` and `src/widgets/*/logic/**`: lines/functions/statements 95%, branches 90%.
  - Global: lines 80%, branches 75%.
- axe-core checks (C1) are added in T5.1 (`@axe-core/playwright`, approved, section 8).

## 7. Tooling

TypeScript:
- `tsconfig.app.json` (`include: ["src"]`): `strict`, `noUncheckedIndexedAccess`, `noImplicitOverride`, `noFallthroughCasesInSwitch`, `noUnusedLocals`, `noUnusedParameters`, `verbatimModuleSyntax`, `isolatedModules`, `moduleResolution: "bundler"`, `jsx: "react-jsx"`, `noEmit`, `paths: { "@/*": ["./src/*"] }`.
- `tsconfig.node.json` (`include: ["vite.config.ts", "playwright.config.ts", "e2e/**/*.ts"]`, `types: ["node"]`): same strict flags, no DOM-only settings. `tsc -b` therefore typechecks e2e specs. No `allowJs`; `eslint.config.js` is not in any project.

ESLint 9 flat config (`eslint.config.js`), in order:
- `js.configs.recommended`, `tseslint.configs.recommendedTypeChecked` with `languageOptions.parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname }`.
- `{ files: ['**/*.js'], ...tseslint.configs.disableTypeChecked }`.
- react-hooks flat recommended config for the installed major (v7: `reactHooks.configs.flat.recommended`; check the plugin README at install).
- `jsxA11y.flatConfigs.recommended`; `jsxA11y.flatConfigs.strict` for `src/components/**`, `src/pages/**`, `src/widgets/**`.
- AC3: `no-eval`, `no-new-func`: error. Base `no-implied-eval` stays off; `@typescript-eslint/no-implied-eval` (in recommendedTypeChecked) covers it. `@typescript-eslint/no-explicit-any`: error; a justified `any` uses `// eslint-disable-next-line @typescript-eslint/no-explicit-any -- <reason>`. `linterOptions.reportUnusedDisableDirectives: 'error'`.
- C6 (all `src/**`): `no-restricted-globals`: `localStorage`, `sessionStorage`, `indexedDB`, `cookieStore`. `no-restricted-properties`: `document.cookie`, and `window`/`globalThis` × `localStorage`/`sessionStorage`/`indexedDB`/`cookieStore`.
- Import-isolation overrides. In flat config a later matching entry replaces the whole options of the same rule (no merge), so these four entries come after everything above, in exactly this order, and each later entry's `no-restricted-imports` list is a full superset of every earlier entry that can match the same file. All use `no-restricted-imports` with `patterns: [{ group: [...], message }]` (gitignore-style, matched against the import string). Shorthand used below:
  - `APP = ['@/widgets', '@/widgets/*', '@/app', '@/app/*', '@/pages', '@/pages/*']`
  - `UI = ['react', 'react/*', 'react-dom', 'react-dom/*', 'react-router', 'react-router/*', '@/components', '@/components/*', '@/hooks', '@/hooks/*']`
  1. Widget top-level files: `files: ['src/widgets/*/*']`, `ignores: ['src/widgets/registry.ts', 'src/widgets/types.ts', 'src/widgets/__tests__/**']` (`src/widgets/*/*` matches `src/widgets/__tests__/registry.test.ts`, which must import `../registry`). Group: `[...APP, '..', '../*']`.
  2. Widget subfolder files: `files: ['src/widgets/*/*/**/*']` (same `ignores`). Group: `[...APP, '../..', '../../*']` (so `components/` and `__tests__/` may import `../index`, `../logic/x`).
  3. Icon: `files: ['src/widgets/*/icon.tsx']` (overlaps 1 only). Group: `[...APP, '.', './*', '..', '../*']` (all relative imports banned; `import type` from `react` stays allowed).
  4. Logic, last: `files: ['src/lib/**/*', 'src/widgets/*/logic/**/*']` (overlaps 2 only). Group: `[...APP, ...UI, '..', '../*', '../..', '../../*']`. Consequences: widget `logic/` imports only `./x` (same folder) and `@/lib/...`, never `../index` or `../components/X`; `src/lib` has no relative imports leaving the current folder, so cross-folder lib imports (including `src/lib/**/__tests__` importing their subject) use `@/lib/...`. The same entry sets `no-restricted-globals` to the full C6 list plus `window`, `document`, `globalThis`, `self` (it replaces the global C6 entry).
- T0.3 must prove the isolation rules fire (gitignore-style relative patterns are easy to get wrong) and record the command and output in the PR description. With `npx eslint --stdin --stdin-filename <path>` (if the project service rejects a stdin path not on disk, write the snippet to that path, lint it, delete it):
  - `src/widgets/calculator/logic/x.ts` importing `../index`, `react`, `@/components/Button`, `../../unit-converter/logic/convert`, `../../../components/Button`: expect one error each.
  - `src/widgets/calculator/index.tsx` importing `../unit-converter`, `@/widgets/registry`: expect errors. `src/widgets/calculator/icon.tsx` importing `./logic/keymap`: expect an error.
  - `src/lib/number/x.ts` importing `../other/y`, `react`: expect errors.
  - No errors for: `./keymap` and `@/lib/number/format` in `logic/x.ts`; `../logic/engine` in `src/widgets/calculator/__tests__/x.test.ts`; `@/components/Button` in `src/widgets/calculator/index.tsx`; `../registry` in `src/widgets/__tests__/registry.test.ts`.
- No stylistic rules, so Prettier needs no ESLint bridge.

Prettier: `{ "singleQuote": true, "trailingComma": "all", "printWidth": 100 }`. Ignore `dist`, `coverage`, `playwright-report`, `test-results`.

Tailwind v4 via `@tailwindcss/vite`; CSS-first config in `src/index.css` (no `tailwind.config.js`):
```css
@import "tailwindcss";
@theme inline {
  --color-accent: var(--color-indigo-600);        /* text, links, primary buttons */
  --color-accent-hover: var(--color-indigo-700);
  --color-accent-contrast: var(--color-white);    /* text on accent fill */
}
```
- v4 colours are oklch (`indigo-600` = `oklch(51.1% 0.262 276.966)`). My sRGB estimate is ≈6.4:1 on white (unverified). T0.3 must measure the rendered accent against `white` and `slate-50` in a browser contrast checker and record the result in DECISIONS.md; required >= 4.5:1 both ways (accent text on white, white on accent).
- Surfaces `white`/`slate-50`; primary text `slate-900`; secondary text minimum `slate-600`. `slate-500` is for control borders only (3:1, SC 1.4.11); `slate-200`/`slate-300` decorative only.
- Focus: `focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-accent` on every interactive element (in shared `Button`). Headings focused programmatically (`tabIndex={-1}`) get `outline-none`.
- No `dark:` classes in v1 (dark mode is v1.1).

npm scripts:
```
dev          vite
build        tsc -b && vite build
preview      vite preview
typecheck    tsc -b
lint         eslint . --max-warnings 0
format       prettier --write .
format:check prettier --check .
test         vitest run --coverage
test:watch   vitest
test:e2e     playwright test
```

## 8. Dependencies (proposal)

Caret ranges of the current stable major at install time; commit `package-lock.json`. Any future package outside this table and the standing approval needs a human decision recorded here and in DECISIONS.md before install.

| Package | Runtime / dev | Install in task | Approval status |
|---|---|---|---|
| react, react-dom (19) | runtime | T0.3 | within approved stack |
| react-router (7) | runtime | T0.3 | within approved stack |
| vite, @vitejs/plugin-react | dev | T0.3 | within approved stack |
| typescript, @types/react, @types/react-dom, @types/node | dev | T0.3 | within approved stack |
| tailwindcss, @tailwindcss/vite | dev | T0.3 | within approved stack |
| eslint, @eslint/js | dev | T0.3 | within approved stack |
| prettier | dev | T0.3 | within approved stack |
| typescript-eslint | dev | T0.3 | APPROVED by human 2026-09-30 |
| eslint-plugin-react-hooks | dev | T0.3 | APPROVED by human 2026-09-30 |
| eslint-plugin-jsx-a11y | dev | T0.3 | APPROVED by human 2026-09-30 |
| vitest, @vitest/coverage-v8 | dev | T0.4 | within approved stack |
| @testing-library/react, @testing-library/dom, @testing-library/user-event, @testing-library/jest-dom | dev | T0.4 | within approved stack |
| jsdom | dev | T0.4 | APPROVED by human 2026-09-30 |
| @playwright/test | dev | T0.4 | within approved stack |
| @axe-core/playwright | dev | T5.1 only | APPROVED by human 2026-09-30 — install in T5.1 only |

Standing approval (DECISIONS.md, 2026-09-30): official plugins/integrations and `@types/*` packages of already-approved tools need no separate human approval; the architect still records them here and in DECISIONS.md.

Why these were needed: typescript-eslint is required for ESLint to parse TS at all; react-hooks catches hook-order/stale-deps bugs; jsx-a11y gives static a11y checks for R7/AC6; jsdom is the DOM environment RTL needs (alternative: happy-dom); @axe-core/playwright automates C1 checks.

Rejected, do not add: icon libraries (C5), state libraries (section 4), `vite-tsconfig-paths`, `eslint-config-prettier`, `prettier-plugin-tailwindcss`, decimal.js/big.js (section 5), `classnames`/`clsx`, `react-is` (registry test checks `$$typeof` directly).

## 9. Later additions (not built in v1)

- Auth: a provider wraps `AppLayout`. `WidgetDefinition` may gain optional `requiresAuth?: boolean`, handled once in `router.tsx`. Existing entries stay valid.
- Backend: `src/services/` with a typed fetch wrapper. Engines stay pure; persistence (e.g. history) goes behind an injected `HistoryStore` interface with an in-memory default, so C6 holds until it changes. The storage lint ban is lifted only inside `src/services/`.
- Dark mode (v1.1): `dark:` variants and dark tokens in `@theme`; the accent token layer avoids hard-coded colours.
- Many widgets: Home groups/filters by `category`; `load` enables preloading on hover.
