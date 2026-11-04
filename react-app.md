# Prompt: Align my React app with the office design system and make it modular

## How to use
1. Open the work repo in your AI coding assistant (agent mode, with read access to the repo and `node_modules`).
2. Fill in the 3 placeholders below.
3. Run **Part A**. It only analyses and writes reports. Nothing in the code changes.
4. Review the reports. Mark each decision approved or rejected.
5. Run **Part B** one phase or one feature at a time.

```
APP_ROOT      = <path to my React app, e.g. apps/my-ui/src>
OFFICE_SCOPES = <npm scope(s) of office packages, e.g. @org/ui, or "discover">
REPORT_PATH   = docs/office-alignment/
```

---

## PART A: Analysis only (read-only)

You are a senior React engineer. Your job is to find out what the office packages offer and how my existing app (APP_ROOT) could use them. You will also propose a modular structure for the app. **In this part you do not change any code.**

### Ground rules (they apply to Parts A and B)
1. **Read-only in Part A.** Do not edit, install, upgrade, or delete anything. Write files only under REPORT_PATH.
2. **The layout stays as it is.** "Layout" means: the routes and pages, where each region sits, the grid/flex arrangement, sizes and proportions, responsive behaviour, and the navigation flow. If a change would alter any of these, flag it. Never apply it.
3. **Using the office library is optional.** Recommend an office component only when it really fits. Each item gets one of these decisions: *Adopt / Adapt / Wrap / Keep & restyle / Keep as-is* (defined in Step 3).
4. **Tokens come before components.** Colours, typography, icons, and the AG Grid theme are usually the biggest wins with the least risk. Put them first.
5. **Back every claim with evidence.** For anything you say about an office package, cite the file and export you read (path + symbol). If you can't verify something, label it "unverified". Never guess props or APIs.
6. **No new dependencies.** If something would need a new package or an upgrade, list it with the reason. Do not add it.
7. **Repo conventions win.** If the repo already has conventions (folder names, lint rules, patterns), follow them over the defaults suggested here.

### Step 1: Find the office dependencies
- Read `package.json` (every workspace), the lockfile, `.npmrc` (registries and scopes), and the monorepo/workspace config. List every internal or office package with its version.
- For each package, look inside `node_modules/<pkg>`: `package.json` (exports, peerDependencies, main/module/types), the `.d.ts` typings, README, CHANGELOG, and any docs or Storybook (`*.stories.*`, `storybook-static/`, `docs/`).
- Sort what you find into these categories:
  - **a. Components**: the full list, with key props and composition patterns (slots, compound components, `as`/polymorphic props).
  - **b. Design tokens**: colours (including semantic and status colours), spacing, radius, shadows, z-index, breakpoints. Note how they are exposed (CSS variables, JS/TS objects, a Tailwind preset, SCSS variables).
  - **c. Typography**: font families, type scale, weights, line heights, and how the fonts are loaded.
  - **d. Icons**: format (React components, SVG sprite, or icon font), naming convention, and size/colour props.
  - **e. AG Grid**: version and edition (community or enterprise). Check which theming style is in use: the Theming API (v33+, a theme object passed through the `theme` prop) or legacy CSS themes (`ag-theme-*` classes plus CSS variables). Note what the office package provides (theme, cell renderers, defaults).
  - **f. Setup**: required providers (such as a ThemeProvider), global CSS or resets, fonts, i18n, and wrapper components.
  - **g. Guidelines**: ESLint or Stylelint configs, accessibility rules, usage docs, do/don't pages.
- Check compatibility with my app: React version, TypeScript version, styling engine, peer-dependency conflicts, and **global CSS or resets that could shift my layout**.

**Output:** `REPORT_PATH/01-office-inventory.md`

### Step 2: Inventory my app (APP_ROOT)
- **Components**: routes and pages, the layout shell, and every reusable or one-off component. Give each one's responsibility and line count. Flag files over about 250 lines and files that do more than one job.
- **Styling**: how styling is done (CSS, CSS modules, Tailwind, CSS-in-JS, a UI kit such as MUI or shadcn). List every hard-coded colour, font, font size, spacing, and shadow value, with how often each appears. Use grep for hex/rgb/hsl, `font-family`, `font-size`, and px values.
- **Icons**: the library in use and how icons are rendered.
- **Grids and tables**: how AG Grid is used (columnDefs, custom cell renderers, theme, row heights, pinned columns), plus any other tables.
- **Logic**: where data fetching, state, and business logic live, and whether they are mixed into the UI components.
- **Tooling**: TypeScript strictness, lint, tests, and build.
- **Overlap and conflicts**: where my styling or UI kit overlaps with the office packages, and whether the two can coexist.

**Output:** `REPORT_PATH/02-app-inventory.md`

### Step 3: Mapping matrix
Write one row for every UI element in my app:

| My component | Location | Office equivalent (pkg + export) | Decision | Why | Layout impact | Effort (S/M/L) | Risk |
|---|---|---|---|---|---|---|---|

The decisions mean:
- **Adopt**: the office component replaces mine, with the same layout and behaviour.
- **Adapt**: use the office component, configured with props or slots to match my behaviour.
- **Wrap**: keep a thin local wrapper (e.g. `components/ui/Button.tsx`) that renders the office component. The app then depends on my interface, not the vendor's.
- **Keep & restyle**: there is no suitable office component, or using one would break the layout or behaviour. Keep mine and rebuild it on office tokens, typography, icons, and guidelines, so it looks and behaves like a native office component.
- **Keep as-is**: changing it isn't worth it.

Add separate rows for: colour tokens, typography, icons, the AG Grid theme, and spacing/radius/shadow tokens.

List the gaps in both directions: (1) useful things the office packages offer that I don't use; (2) things I have that the office packages lack.

**Output:** `REPORT_PATH/03-mapping-matrix.md`

### Step 4: Target modular structure
Propose a structure based on React guidance (react.dev): pure components, one responsibility per component, custom hooks for reusable logic, and state kept as close as possible to where it's used. React doesn't mandate a folder layout. Use the common feature-based layout below, adjusted to the repo's existing conventions:

```
src/
  app/               # app shell: providers, router, global styles
  routes/            # thin route/page components that compose features
  features/
    <feature>/
      components/    # UI used only by this feature
      hooks/         # feature logic (useXxx)
      api/           # data fetching, DTO -> view-model mapping
      utils/
      types.ts
      index.ts       # public API; other code imports the feature only through this
  components/
    ui/              # shared presentational components and office wrappers
    layout/          # layout primitives used by the shell (no visual change)
  theme/
    tokens.ts        # single bridge: office tokens -> app semantic tokens
    typography.ts
    agGridTheme.ts
    Icon.tsx         # <Icon name="..."/> adapter over the office icon set
  hooks/             # shared hooks
  lib/               # framework-agnostic helpers, API client
  types/
```

Rules:
- **Dependency direction**: shared code (`components`, `hooks`, `lib`, `theme`) → `features` → `app`/`routes`. A feature never imports another feature's internals. Go through its `index.ts`, or move the shared part up into shared code.
- **Keep UI and logic separate**: components get data through props. Fetching, state, and business rules live in hooks or `api/`.
- **No hard-coded design values** (colours, fonts, px spacing) outside `theme/`.
- One component per file. Keep tests and stories next to the component they cover.
- If the repo has ESLint, propose import-boundary rules (e.g. `import/no-restricted-paths`). Don't install anything.

**Output:** `REPORT_PATH/04-target-structure.md`. Include the proposed tree and a table that maps **every current file → its new path**.

### Step 5: Theming bridge plan
- **Semantic tokens**: define app-level names (e.g. `color.surface`, `color.text.primary`, `color.status.danger`) and map each one to an office token. List every hard-coded value → the token that replaces it. Where there's no exact match, give the nearest token and note the visual difference.
- **Typography**: map each of my text styles to the office type scale, and describe the font-loading setup.
- **Icons**: map each icon I use to an office icon name. Mark any that are missing.
- **AG Grid**: explain how to apply the office grid theme without changing columns, widths, pinned columns, row heights, or density. Flag anything that would change them.
- **App setup**: the providers and global imports needed at the app root, and any reset or CSS conflicts to handle.

**Output:** `REPORT_PATH/05-theming-plan.md`

### Step 6: Summary (read this first)
Write `REPORT_PATH/00-summary.md`, one page at most:
- What the office packages provide, in a few lines
- **Quick wins**: tokens, typography, icons, AG Grid theme
- **Components to adopt, adapt, or wrap**, and **components to keep & restyle**
- **What not to adopt**, and why
- **Risks, unknowns, and open questions** for me or the design-system team
- **Phased plan**, with scope, effort, and how to verify each phase:
  - Phase 1: foundations (theme bridge, providers, structure move). The only visible changes are colours, fonts, and icons.
  - Phase 2: per-feature component alignment
  - Phase 3: cleanup and lint enforcement

Then **STOP**. Do not change any code. Ask me to review the reports.

---

## PART B: Execute (one approved phase or feature at a time)

> Execute **<Phase N / feature X>** from `REPORT_PATH/00-summary.md`, using only the decisions I marked as approved.

Rules:
1. **Baseline first.** Run build, lint, type-check, and tests. If Playwright or Storybook is available, take screenshots of the affected screens. Otherwise, write down the current layout structure.
2. **Structure moves first, with no logic changes.** Move files, fix the imports, and make sure the build passes. Put this in its own commit.
3. **Then make small changes**, one component or one token group per commit.
4. **The layout stays as it is.** Keep the same hierarchy of layout containers, the same grid/flex setup, the same dimensions, and the same breakpoints. If an office component would force a layout change, stop and report it. Never work around it silently.
5. **Keep & restyle components** must use office tokens, typography, and icons, and follow office guidelines (accessibility, naming, prop patterns).
6. **After each step:** run build, lint, type-check, and tests, then compare against the baseline. List every visual difference, however small.
7. **Finish with a change log:** files moved, components replaced, tokens applied, and any open items.
