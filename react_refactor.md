# Refactor this codebase to a clean, consistent, fully tested standard

You are working in GitHub Copilot agent mode on a production React app. Your job: audit the whole `src/` tree, agree a plan with me, then refactor it one unit at a time until every component, hook, provider, atom, API function, model, utility, mock and test meets the standards in §4. **Behaviour must not change.** Start with Phase 1 now.

## 0. Context

- Stack (re-check package.json, don't assume): React 19.3 + React DOM 19.3, TypeScript 5.9.3, Vite (build), Vitest (tests), Jotai 2.20.1 (state), Axios (HTTP), AG Grid 36.3 (grids), LESS CSS modules (one `*.module.less` per component), ESLint flat config (`eslint.config.mjs`), Prettier.
- Office design-system packages: every package in package.json whose name starts with `@TAO-` (components, icons, colors, typography). They are the source of truth for all UI.
- Current `src/` layout: `api/ assets/ components/ features/ hooks/ i18n/ mocks/ models/ provider/ state/ styles/ theme/ utils/`, plus `App.tsx`, `main.tsx`, `routes.tsx`. Tests live in the root `test/` folder (check for colocated tests too).
- Before planning, read every `*.md` file at the repo root (README plus the redesign, styling-upgrade and schema notes). They contain decisions you must respect.

## 1. Ground rules (every phase)

1. **No behaviour change.** Keep the same data, routes, API calls (URLs and payloads), validation outcomes and user flows. The only visual changes allowed come from using an `@TAO-` component as intended in place of a hand-made one. List each one in the unit report so I can check it.
2. **Installed packages only.** Import only packages listed in package.json, never a transitive dependency. If a new package would genuinely help, add it to `PLAN.md → Proposals` with the reason. Do not install it.
3. **Do not touch:** `src/META-INF/`, `src/WEB-INF/`, `src/404.html`, `src/500.html`, `index.html`, `test-runtime.html`, `public/`, `serve.json`, `spa-support-config.json`, `build-info-config.json`, `package-lock.json`, `tsconfig.tsbuildinfo`, `UI-test-agent/`. Change `tsconfig.json`, `vite.config.ts`, `vitest.config.ts` or package.json scripts only when an approved plan item says so.
4. **Time-box library research.** Don't study the `@TAO-` packages, AG Grid or Jotai in general. When you need a specific component or API, read only that export's type definition in `node_modules`; workspace search skips `node_modules`, so use the terminal (`ls`, `grep -r`, `sed -n`). Never use anything marked `@deprecated` in the installed typings.
5. **Ask instead of guessing** when intent is unclear (a business rule, a validation rule, which of two behaviours is right). Put the question under "Open questions" and skip that item.
6. **Git:** create branch `refactor/clean-code` from my current branch. If the working tree is dirty, stop and tell me. Commit locally after each unit. Never push or rewrite history.
7. **Progress lives in files, not chat.** `docs/refactor/PLAN.md` is the source of truth. A new chat session resumes from the first unchecked item.

## 2. Phase 1: Audit (read-only, no code changes)

Record a baseline. Run typecheck, lint, tests and build using the package.json scripts (typecheck is `npx tsc --noEmit`, or `tsc -b` if the project uses build mode). Note any failures that already exist; don't fix them yet.

Write `docs/refactor/AUDIT.md` containing:
- The baseline results.
- An inventory per shared folder and per feature: files, lines, components, hooks, atoms, API functions, models, utils, tests and mocks.
- Findings with file paths and counts, grouped as follows:
  - **Styling in TS/TSX:** `style={…}`, style objects or constants, color/px literals, class names built from strings, AG Grid `cellStyle`/`rowStyle`/`getRowStyle`; unused or duplicated LESS; hard-coded colors or fonts in LESS; `!important`; `:global` overrides of `@TAO-` internals.
  - **`@TAO-` usage:** raw HTML or hand-made components where an `@TAO-` equivalent exists, other icon sources, wrappers or re-exports of `@TAO-` components, deep imports into package internals.
  - **Clean code:** unclear names, long components or functions, comments that restate the code, commented-out code, dumping-ground files (`helpers.ts`, `common.ts`), duplication, magic values, nested ternaries in JSX.
  - **Types:** `any`, `as` casts on external data, non-null `!`, `@ts-ignore`, enums, duplicated types.
  - **State/API:** API calls inside components, `axios` imported outside `src/api`, fetching in `useEffect` without cancellation, server data copied into several atoms, `atomFamily` usages (deprecated in Jotai 2.20).
  - **Validation:** places where user input or API data is used unvalidated; every validation approach in use today.
  - **Tests:** untested logic per folder; test-style problems.
  - **Mocks:** how mocking works today (mechanism, how scenarios are switched), endpoints with no mock, missing scenarios.
  - **Placement:** feature-only code in shared folders, shared code duplicated across features, cross-feature imports of internals, overlap between `styles/` and `theme/`.
- Stricter TS flags, measured but not applied: for each `strict` sub-flag that is off, and for `noUncheckedIndexedAccess`, report the error count from `npx tsc --noEmit --<flag>`.

## 3. Phase 2: Plan and rules, then STOP

Write:
1. **`docs/refactor/PLAN.md`:** an ordered checklist of work units. Unit 0 is the shared foundations (§5). Then one unit per feature (split large features), then cleanup of the shared folders, then lock-in (§6). For each unit give its scope (folders), what changes, risks, tests to add and a rough size. End with two sections: `Proposals (need approval)` and `Open questions`.
2. **`docs/refactor/DECISIONS.md`:** the decision log, in the format below. Pre-fill it with the decisions the audit forces: the single validation approach, the mock mechanism, test file location, the `styles/` vs `theme/` split, and each existing `@TAO-` wrapper (keep it with a reason, or remove it).
3. **`.github/copilot-instructions.md`:** the standing rules for all future Copilot work. It is a condensed version of §4 tailored to this repo with real package names and paths: 150 lines at most, rules rather than essays. If the file exists, merge into it and list any conflicts for me.

Then **stop** and show me the top findings (with numbers), the unit list, the proposals and the open questions. Wait for my approval.

Decision record format:
```
## D-001 <short title>
- Date / Status: <date> / proposed | accepted
- Context: the problem
- Decision: what we do
- Reasoning: why; alternatives considered and why they were rejected
- Consequences: what it costs, what to watch
```

## 4. Standards

### 4.1 Styling: TS/TSX carries no styling logic
- A TS/TSX file only *names* styles. Every visual decision (color, spacing, size, layout, states, responsive rules, animation, overrides) lives in the component's `ComponentName.module.less`.
- Allowed in TSX:
  - `className={styles.summaryCard}`.
  - State and variants as data attributes (`data-status={status}`, `data-selected={isSelected}`), styled in LESS with `.row[data-status='error']`.
  - At most one conditional class via `cx(styles.row, isSelected && styles.rowSelected)`.
  - `@TAO-` component props that are part of its API (`variant`, `size`, …).
- Not allowed in TS/TSX:
  - `style={{…}}`, style objects or constants, CSS-in-JS.
  - Color or size literals (`'#fff'`, `'12px'`, `rgba(…)`).
  - Class names built with template literals or concatenation.
  - AG Grid `cellStyle`, `rowStyle` or `getRowStyle`.
- The one exception is a value known only at runtime (a progress percentage, a measured width). Pass it as a CSS custom property through a `cssVars` helper, `style={cssVars({ progress: `${percent}%` })}`, and read it with `var(--progress)` in LESS.
- LESS rules:
  - Class names are camelCase, so they are used as `styles.summaryCard`.
  - Colors, fonts, spacing, radii and shadows come from `@TAO-` tokens (LESS variables or CSS custom properties, whichever they expose), or from `src/styles/` variables that map to them. No raw hex, rgb or font values.
  - Nest at most 3 levels deep. No `!important`.
  - Use `:global(…)` only to reach third-party internals, each with a one-line "why" comment. Reaching into `@TAO-` internals also needs a decision record.
  - Delete unused classes. Shared variables and mixins live in `src/styles/` and are imported, never copied.
- One `.module.less` per component, with the same base name, imported as `styles`.

### 4.2 `@TAO-` libraries: use them as-is
- Import components, icons, colors and typography straight from each package's public entry point. Never deep-import `dist/` or internal paths.
- Every UI element an `@TAO-` package provides comes from that package: buttons, inputs, selects, checkboxes, dialogs, tooltips, notifications, typography, icons, layout primitives. That means no raw `<button>`, `<input>`, `<h1>` etc. where an equivalent exists, no other icon sets, and no inline `<svg>` copies of `@TAO-` icons.
- Interactions (hover, focus, disabled and loading states, keyboard handling, open/close) come from the components' own props. Never re-implement them.
- **No wrappers:**
  - No `MyButton` that just forwards props to `Button`.
  - No barrels that re-export `@TAO-` components.
  - No layers that rename or remap props.
  - No CSS that overrides the components' look.
- Composing several `@TAO-` components into a domain component (`InvoiceSummaryCard`) is normal feature code, not a wrapper.
- A wrapper is allowed only in two cases: the same non-trivial behaviour is needed in 3+ places and props or composition can't express it, or an integration requires it (for example, binding to the form or validation layer). **Before writing one, add a decision record** with context, reasoning, alternatives and consequences. Name the wrapper for what it adds (`CurrencyAmountField`, not `CustomInput`). The same applies to any unavoidable style override of an `@TAO-` component.
- For each existing wrapper, either remove it (use the `@TAO-` component directly) or keep it with a decision record.
- If a needed capability seems missing, log it under "`@TAO-` gaps" in `DECISIONS.md` rather than quietly building a workaround.

### 4.3 Where code lives (the common library)
- The top-level folders `src/api`, `components`, `hooks`, `models`, `provider`, `state`, `utils`, `styles`, `theme`, `i18n` and `mocks` form the **shared (common) layer**. Code goes there only when 2 or more features use it.
- Code used by one feature lives in `src/features/<feature>/`. It uses the same subfolders as needed (`components/ hooks/ api/ state/ models/ utils/ validation/`), plus an `index.ts` that is the feature's public API.
- Imports point one way, features → shared. Shared code never imports from a feature. A feature imports another feature only through that feature's `index.ts`.
- Component folder layout: `ComponentName/ComponentName.tsx`, `ComponentName.module.less`, the test file (at the decided location), and optionally `useComponentName.ts`, `componentName.columns.ts` (AG Grid) and `index.ts`.
- No dumping-ground files (`helpers.ts`, `common.ts`, `misc.ts`, a giant `constants.ts`). Split them by purpose (`formatCurrency.ts`, `invoiceStatus.ts`).
- Move code unit by unit, never all at once. If an import alias such as `@/` exists, use it everywhere. If none exists, propose one.

### 4.4 Clean code: the code explains itself
- Names state intent in domain words: `fetchOpenInvoices`, `mapInvoiceDtoToInvoice`, `isInvoiceOverdue`, `canSubmitPayment`, `selectedInvoiceIdAtom`, `useInvoiceFilters`, `InvoiceStatusBadge`. Event props are `onX` and handlers `handleX` (`handlePaymentSubmit`). Booleans read as questions (`is`/`has`/`can`/`should`).
- Banned names:
  - Generic names: `data`, `item`, `obj`, `temp`, `res`, `val`, `flag`, `list2`.
  - Abbreviations: `acc`, `amt`, `btn`.
  - `handleClick` when there is more than one click.
  - Files called `utils.ts` or `helpers.ts`.
- Code that orchestrates reads as a sequence of well-named steps, with the high-level function first and details below it. Each function does one job and returns early.
- Control flow:
  - No nested ternaries.
  - No boolean parameters that switch behaviour; split the function or take an options object.
  - No magic numbers or strings; use named constants.
- A component reads top to bottom: hooks → derived values → handlers → JSX. The JSX reads like a description of the screen. Pull conditions out into named variables (`const showEmptyState = …`) and complex or repeated blocks into subcomponents.
- Comments explain only *why*: a business rule, a workaround, a non-obvious constraint. Delete comments that repeat the code, commented-out code and stale TODOs (list any remaining TODOs in the report).
- Size guidelines: a component around 150 lines, a function around 30, a file around 250. Split anything larger.
- Use named exports. Use default exports only where a tool needs one (for example, `React.lazy`). Write components as function declarations.

### 4.5 Components, hooks, providers (React 19.3)
- Props:
  - Type them (`type InvoiceTableProps = {…}`) and destructure them in the signature.
  - No `React.FC`, no `any`.
  - Use `ComponentProps<typeof X>` when passing props through to an `@TAO-` component.
- React 19 idioms:
  - `ref` is a normal prop. Write no new `forwardRef`, and convert existing ones in any file you touch.
  - Render `<SomeContext value={…}>` instead of `<SomeContext.Provider>`.
- Logic lives in hooks and markup in components. A component with more than about 2 state/effect hooks, or with any data access, gets a `useXxx` hook.
- Loading, error, empty and success each render explicitly (early returns), using `@TAO-` feedback components. Text comes from i18n.
- Compute derived values during render; don't sync them with `useEffect` + `useState`. Use `useEffect` only to sync with external systems. For non-reactive logic inside an effect, use `useEffectEvent` instead of disabling `exhaustive-deps`.
- Use `useMemo`, `useCallback` and `memo` only where they matter (AG Grid props, context values, expensive computations). Don't add them everywhere.
- List keys are stable ids, never array indexes.
- No hard-coded user-facing text. Use the existing `src/i18n` setup and add new keys to the default locale. List missing translations for other locales in the report; never invent translations.
- Hooks: named `use` + intent, one responsibility each, typed return value. Clean up subscriptions and timers, and abort requests.
- Providers:
  - Compose all app providers in one `AppProviders` component.
  - Each context gets a typed `useX()` hook that throws a clear error when used outside its provider.
  - Memoize the context value.
  - Prefer Jotai atoms over Context for shared app state; use Context to inject configuration and services.

### 4.6 State: Jotai 2.20.1
- Atom names end in `Atom`. Feature atoms live in `features/<feature>/state/`, app-wide atoms in `src/state/`.
- Keep primitive atoms minimal and derive everything else with read-only atoms. Write multi-step updates as write-only action atoms (`submitPaymentAtom`).
- Use `useAtomValue` to read and `useSetAtom` to write. Use `useAtom` only when a component needs both.
- Server data lives in async atoms:
  - Pass the read `signal` through to the API function: `atom(async (get, { signal }) => fetchInvoices(get(invoiceFiltersAtom), { signal }))`.
  - Use `loadable` or `unwrap` where Suspense isn't used, and `atomWithRefresh` for refetching.
  - Never copy server data into extra atoms.
- `atomFamily` from `jotai/utils` is deprecated in 2.20. Add no new usages and list the existing ones. Moving to the `jotai-family` package needs a new dependency, so it goes under Proposals.
- Never create atoms during render without `useMemo`.
- Tests use a fresh `createStore()` per test.

### 4.7 API layer and models (Axios)
- One Axios instance in `src/api/` (`httpClient`). It takes base URL and headers from `import.meta.env` and uses interceptors for auth headers and to turn every failure into a typed `ApiError` (`status`, `code`, `message`, `fieldErrors?`). No other file imports `axios`.
- One module per domain: `features/<feature>/api/invoiceApi.ts`, or `src/api/` if shared. It contains plain typed functions such as `fetchInvoices(params, { signal }): Promise<Invoice[]>`.
  - They return domain models, never `AxiosResponse`.
  - They accept an `AbortSignal`; no `CancelToken`.
  - URLs are named constants or builders, and stay exactly as today.
- Keep wire types and app types separate only when they differ: `InvoiceDto` → `mapInvoiceDtoToInvoice` → `Invoice`. Mapping and validation happen at this boundary.
- Components never call the API directly; they go through hooks or atoms.
- Models:
  - Use `type` aliases consistently.
  - In new or refactored code, use string-literal unions or `as const` objects instead of enums.
  - Derive types with `Pick`/`Omit` instead of duplicating them.
  - Use `unknown` plus type guards instead of `any`, and never cast external data with `as`.

### 4.8 Validation wherever data enters
- Validate at every entry point:
  - Form input (per field and on submit).
  - API responses the UI relies on (in the mapper).
  - Request payloads before sending.
  - Route and query params.
  - Env config at startup.
  - Persisted values (localStorage, `atomWithStorage`).
- Use one approach across the app:
  - If a validation library is already in package.json, use it.
  - Or use the `@TAO-` form components' built-in validation, if they have it.
  - Otherwise, write small pure validator functions and type guards (`isValidIban`, `parseInvoiceDto`).
  - Record the choice in `DECISIONS.md`.
- Keep every existing rule; never weaken or drop one. Match the backend rules in the schema notes. If a rule is unclear, add it to Open questions.
- Show validation errors through the `@TAO-` field and feedback components with i18n messages. Map API `fieldErrors` onto the same fields.
- Validators are pure and fully unit tested: valid, invalid and boundary cases.

### 4.9 Mocks: every scenario works offline
- Keep the existing mock mechanism in `src/mocks`; find out first whether it is MSW, an Axios adapter or interceptor, or static JSON. Don't introduce a new one.
- Every endpoint the app calls has a mock. Fixtures come from typed factory functions (`buildInvoice(overrides?)`) typed against the DTOs, so a model change breaks stale mocks at compile time.
- Each endpoint has named scenarios:
  - Success (typical), empty, single item, large list or pagination.
  - Boundary data: long text, zero/negative/max amounts, special characters, missing optional fields.
  - Validation error (400 with field errors), 401, 403, 404, 409 where relevant, 500.
  - Network error or timeout, and a slow response so loading states are visible.
- One switch selects the scenario offline (env var, URL parameter or localStorage, whichever fits the current mechanism). Document it with the scenario list in `src/mocks/README.md`. The whole app must run offline in mock mode; if no script for that exists, propose one.
- Mocks never ship in the production bundle; guard them with Vite env flags.
- Tests reuse the same fixtures and scenarios, so there is one source of truth. Tests never hit the real network.

### 4.10 Unit tests (Vitest)
- Use what's installed (Vitest, Testing Library, user-event, jest-dom, MSW, …; check package.json) and the decided test location.
- Test every piece of logic:
  - utils, mappers and validators (table-driven with `it.each`)
  - atoms (with a fresh store)
  - hooks (`renderHook`)
  - API functions (against the mock scenarios: success plus each error type)
  - components (each state, loading/empty/error/success, plus user interactions)
- Test behaviour, not implementation. Query by role, label or text the way a user would. No assertions on CSS class names or `@TAO-` internals, and no snapshot-only tests.
- Test names are sentences: `it('shows the overdue badge when the due date has passed')`. Use Arrange-Act-Assert, one behaviour per test, and no logic in tests.
- Shared test helpers live in one place, for example `renderWithProviders` with a fresh Jotai store, i18n and the router.
- Before refactoring untested code, first add tests that pin its current behaviour, then refactor with them passing.
- If a coverage tool is installed, report coverage per feature. Aim for at least 80% of lines on utils, mappers, validators, state, hooks and API, with meaningful tests rather than padding.

### 4.11 TypeScript 5.9
- Never:
  - `any`
  - a non-null `!`
  - `@ts-ignore` (`@ts-expect-error` with a reason only if unavoidable)
- Always:
  - `import type` for type-only imports
  - `satisfies` for config and lookup objects
  - an exhaustive `switch` with a `never` check
  - explicit return types on exported functions and hooks
- Leave the `tsconfig.json` flags alone. Propose changes based on the Phase 1 measurements.

### 4.12 AG Grid 36.3
- Column definitions live in `<grid>.columns.ts`, typed `ColDef<Row>[]`. Define them outside the component, or memoize them if they depend on props. Keep `defaultColDef` and callbacks stable, and set `getRowId`.
- Format values with `valueFormatter`/`valueGetter` (functions in utils, with tests). Use React cell renderers (typed `CustomCellRendererProps<Row, Value>`) only for interactive or `@TAO-` content.
- Style with `cellClass`, `cellClassRules` and `rowClassRules` pointing at `styles.*`, never with `cellStyle`, `rowStyle` or `getRowStyle`.
- Use one grid theme for the whole app: the Theming API built from `@TAO-` color and typography tokens, or a theme `@TAO-` already provides. Register modules in one place (`AgGridProvider` or `ModuleRegistry`, whichever the app already uses, never both). No per-grid look-and-feel CSS.
- Use no grid option marked `@deprecated` in the installed typings.

## 5. Phase 3: Execute one unit at a time

**Unit 0: shared foundations.** Build only what's missing:
- `httpClient` and `ApiError` (§4.7)
- the `cssVars` helper, and a `cx` helper only if neither `clsx` nor `classnames` is in package.json
- validation building blocks (§4.8)
- the mock scenario switch and fixture-builder pattern (§4.9)
- test helpers (§4.10)

For each unit:
1. Re-read the unit in `PLAN.md`, plus `DECISIONS.md` and `.github/copilot-instructions.md`.
2. Pin behaviour. Add the missing tests for the current code and get them passing.
3. Refactor in this order: models → api (mappers, validation) → mocks → state → hooks → components → LESS.
4. Move code to the right layer (§4.3), update imports, and delete dead code and unused LESS.
5. Verify with typecheck, lint on changed files, Prettier, related tests, the full test suite and the build. Everything must pass with no new warnings before you move on.
6. Check the Definition of Done for every file you touched.
7. Tick the unit in `PLAN.md`, update `DECISIONS.md`, and commit as `refactor(<scope>): <what changed>`.
8. Report in the format below, then stop. When I reply "continue", do the next unit. "run N units" means do N units, then stop.

**Definition of Done (per file):**
- [ ] No styling in TS/TSX beyond `styles.*`, data attributes, one `cx` and `cssVars`.
- [ ] `@TAO-` components are used directly; every wrapper or override has a decision record.
- [ ] Names explain intent; no comments that restate code; no dead or commented-out code.
- [ ] The file is in the right layer and its imports follow the dependency direction.
- [ ] Types are strict: no `any`, `!` or unchecked `as`.
- [ ] Inputs and API data are validated, and every existing rule still holds.
- [ ] Mocks cover all scenarios for the endpoints touched.
- [ ] Tests cover the behaviour and pass, and the build passes.

## 6. Phase 4: Lock it in

Add rules to `eslint.config.mjs` using only ESLint core rules and plugins that are already installed:
- `no-restricted-syntax`:
  - JSX `style` attributes, except `style={cssVars(…)}`.
  - Color literals in TS/TSX.
  - Raw HTML elements that have an `@TAO-` equivalent (from the audit).
- `no-restricted-imports`, using `files` globs per folder:
  - `axios` outside `src/api/`.
  - Deep imports into `@TAO-` packages.
  - Other icon libraries.
  - One feature's internals from another feature.
  - Anything under `features/` from the shared folders.
- typescript-eslint, if installed: `no-explicit-any`, `consistent-type-imports`, `no-non-null-assertion`, `ban-ts-comment`.
- The eslint-plugin-react-hooks recommended rules, including its compiler-based rules such as `set-state-in-effect` if the installed version ships them.

Set each rule to `error` once the codebase passes it. Otherwise set it to `warn` and list the remaining counts.

Finally, write `docs/refactor/REPORT.md` with:
- Before/after numbers against the audit.
- All decisions.
- The `@TAO-` gaps.
- The visual changes to check.
- The proposals.
- The remaining debt.

## Report format (after each phase or unit)

```
Unit: <name> — DONE | BLOCKED
Changed: <n> files | moved: <…> | deleted: <…>
Checks: typecheck ✔ lint ✔ tests ✔ (<n> passed, +<n> new) build ✔
Decisions: D-00x <title>, …
Visual changes to verify: <components/screens, or "none">
Open questions: <…>
Next unit: <name>
```
