# Task: Standardize this repo's GitHub Copilot customization files

You are GitHub Copilot in agent mode in IntelliJ IDEA. Bring the Copilot customization files under `.github/` in line with the standard below. Work in four phases and stop after Phase 2 until I reply "approved".

## Scope
- In scope (read and change only these): `.github/copilot-instructions.md`, `.github/instructions/`, `.github/prompts/`, `.github/agents/`, `.github/chatmodes/`, `.github/skills/`.
- Out of scope (do not open or change): everything else, including `.github/workflows/`, `CODEOWNERS`, issue/PR templates, `dependabot.yml`, source code and build files.
- Do not commit or create branches.

## Ground rules
1. Preserve meaning. You may reorganize, convert, deduplicate and tighten wording. Never invent rules, facts, commands, versions or paths.
2. If a template section needs a fact you don't have, leave the section out and ask me.
3. If two files contradict each other, don't choose. Ask me.
4. Each rule lives in exactly one file. Agents, prompts and skills must not restate coding rules, because instructions files are applied automatically.
5. Files must work in Copilot for IntelliJ and in Copilot on GitHub.com (cloud agent, code review). Use no VS Code-only keys or syntax.
6. Rename and move files (with `git mv` if available) instead of deleting and recreating them, so history is kept. Update every reference to a renamed file.
7. Comments in the YAML examples below are explanations. Don't copy them into files.

## Target layout
~~~text
.github/
├── copilot-instructions.md        # repo-wide, always applied
├── instructions/
│   └── <scope>.instructions.md    # applied automatically to files matching applyTo
├── prompts/
│   └── <verb>-<object>.prompt.md  # run by hand in chat as /<verb>-<object>
├── agents/
│   └── <role>.agent.md            # picked from the agent dropdown
└── skills/
    └── <skill-name>/
        ├── SKILL.md               # loaded automatically when its description matches the task
        ├── references/            # optional: detailed docs SKILL.md links to
        ├── scripts/               # optional: scripts the skill runs
        └── assets/                # optional: templates, sample files
~~~
`.github/chatmodes/` and all `*.chatmode.md` files are legacy (chat modes are now custom agents). None may exist when you finish.

## Where content belongs
| Content | File |
|---|---|
| True for every task in the repo | `copilot-instructions.md` |
| True only for certain files or folders | `instructions/<scope>.instructions.md` with `applyTo` |
| A task I start by hand, same steps every time | `prompts/<verb>-<object>.prompt.md` |
| A role with its own workflow and tool limits, used for a whole conversation | `agents/<role>.agent.md` |
| A procedure Copilot should load by itself when relevant, optionally with scripts, templates or reference docs | `skills/<skill-name>/SKILL.md` |

## Rules for all files
- Names: lowercase kebab-case (`a-z`, `0-9`, `-`). Only exceptions: `SKILL.md`, `copilot-instructions.md`.
- Frontmatter: YAML between `---` lines at the very top. Double-quote every string (a glob starting with `*` is invalid YAML unquoted). Inline lists: `["read", "search"]`.
- Keys: only the keys listed for that file type. Convert legacy keys as specified. If a key isn't mentioned anywhere in this prompt, keep it and ask me.
- Body: one `#` title, then `##` sections in the order given. Short, imperative, checkable bullets ("Use constructor injection", not "Write clean code"). No generic advice Copilot already follows.
- Links: relative Markdown links, e.g. `[Java rules](../instructions/java.instructions.md)`.
- No VS Code-only syntax: `#tool:`, `#file:`, `${input:…}`, `${selection}`, `${file}`, `${workspaceFolder}`. When a prompt needs input, write "If the user hasn't provided X, ask for it."
- No secrets, tokens, credentials or personal data.

## File types

### 1. `.github/copilot-instructions.md`
- No frontmatter. Max 4,000 characters (Copilot code review reads only the first 4,000; the file is sent with every chat request).
- Sections, in this order (omit any you have no facts for): `# <Project name>`, `## Overview` (2–4 sentences: what it is, who uses it), `## Tech stack` (languages, frameworks, build tool, with versions), `## Build and test` (exact commands), `## Project structure` (key folders and their purpose, max 10 lines), `## Conventions` (only rules that differ from framework defaults), `## Boundaries` (three lists: **Always**, **Ask first**, **Never**).
- If the file doesn't exist, create it only when other in-scope files contain repo-wide rules. Otherwise ask me.

### 2. `.github/instructions/<scope>.instructions.md`
~~~yaml
---
description: "Java coding rules"          # required
applyTo: "src/main/java/**/*.java"        # required; several globs comma-separated in one string
excludeAgent: "code-review"               # optional: "code-review" or "cloud-agent"; omit to apply everywhere
---
~~~
- One topic per file, max 4,000 characters (aim for 20–60 lines). Body: `# <Topic>`, then bullets grouped under `##` headings.
- Change `excludeAgent: "coding-agent"` to `"cloud-agent"`. Remove `name`.
- Check every `applyTo` glob against the repository file list (paths only; don't open files). If a glob matches nothing, ask me.
- A file with `applyTo: "**"` holds repo-wide rules: merge it into `copilot-instructions.md`, tightening wording to stay within 4,000 characters.
- A file without `applyTo` is never applied automatically: propose a real `applyTo` or converting it to a skill.

### 3. `.github/prompts/<verb>-<object>.prompt.md`
~~~yaml
---
description: "Create a REST endpoint with tests"   # required
agent: "agent"                                     # optional: "ask", "agent", "plan" or the name of an agent in .github/agents/
tools: ["read", "search", "edit"]                  # optional; allowed tool names only (see Tool names)
---
~~~
- 20–80 lines. Body: `# <Title>`, `## Goal`, `## Inputs` ("If not given in the chat message, ask for: …"), `## Steps` (numbered), `## Output` (exact format), `## Done when` (checkable criteria).
- Change `mode:` to `agent:` (`ask` stays `ask`; `edit` and `agent` become `agent`).
- Remove `model` and `argument-hint`. If `name` differs from the file name, rename the file to the kebab-case `name`, then remove `name`.

### 4. `.github/agents/<role>.agent.md`
~~~yaml
---
name: "code-reviewer"                           # required; equals the file name without .agent.md
description: "Reviews changes for bugs and convention violations. Use before opening a PR."   # required
tools: ["read", "search"]                       # optional; omit = all tools, [] = no tools
user-invocable: false                           # optional; only to hide a helper agent from the dropdown
disable-model-invocation: true                  # optional; only to stop Copilot invoking it automatically
---
~~~
- 40–150 lines, hard limit 30,000 characters. Body: `# <name>`, a 1–2 sentence role statement, `## When to use`, `## Workflow` (numbered), `## Boundaries` (**Always** / **Ask first** / **Never**), `## Output format`.
- If `name` is missing, set it to the file name. If it differs, rename the file to the kebab-case `name` and make them equal.
- Change `infer: false` to `disable-model-invocation: true`; remove `infer: true`. Replace `tools: ["*"]` by removing `tools`.
- Remove `model`, `handoffs`, `argument-hint`, `advancedOptions`.
- Keep `mcp-servers`, `metadata` and `target` unchanged (used only by GitHub.com's cloud agent) and list them in the report.

### 5. `.github/skills/<skill-name>/SKILL.md`
~~~yaml
---
name: "database-migrations"   # required; equals the folder name; 1–64 chars of a-z, 0-9 and single hyphens, no hyphen at start/end
description: "Creates and validates Flyway migrations. Use when adding or changing database tables, columns or indexes."   # required; max 1,024 chars; what it does AND when to use it
license: "Apache-2.0"         # optional
---
~~~
- Body under 500 lines (~5,000 tokens): `# <Title>`, `## When to use`, `## Steps`, `## Examples` (input → output), `## Gotchas`, `## Resources`.
- Move long detail into `references/`, `scripts/` or `assets/`, linked from SKILL.md with relative paths, one level deep.
- If the folder name differs from `name`, rename the folder.
- Keep `compatibility` and `metadata` if present. Keep `allowed-tools` and `disable-model-invocation` if present and list them in the report.

## Migrating legacy files
- Each `.github/chatmodes/<x>.chatmode.md` becomes `.github/agents/<x>.agent.md` following rule 4; delete `chatmodes/` once empty. If a chat mode only describes a single task, propose a prompt file instead.
- Rename plain `.md` files in `agents/` to `.agent.md`.
- **Tool names.** Allowed: `read`, `edit`, `search`, `execute`, `web`, `agent`, `todo`, and MCP tools as `<server>/<tool>` or `<server>/*`. Convert legacy names, then remove duplicates:
  - `read`: readFile, problems, changes, testFailure
  - `edit`: editFiles, createFile, createDirectory, editNotebook
  - `search`: codebase, search, fileSearch, textSearch, usages, findTestFiles, searchResults
  - `execute`: runCommands, runInTerminal, getTerminalOutput, terminalLastCommand, terminalSelection, runTasks, runTests
  - `web`: fetch
  - `agent`: runSubagent
  - `todo`: todos
  - Namespaced names like `read/readFile` or `execute/runInTerminal` become their prefix.
  - Remove VS Code-only tools (`vscodeAPI`, `extensions`, `new`, `openSimpleBrowser`, `githubRepo`, `vscode/*`) and any name you can't map; list them in the report.

## Phases
**Phase 1: Inventory (read only).** One table row per in-scope file: path | type | frontmatter keys | characters | lines | problems.

**Phase 2: Plan, then STOP.** A table: current path | action (keep / edit / rename / move / convert / merge / split / delete) | new path | reason. Then a numbered **Questions** list (missing facts, contradictions, unknown keys, globs matching nothing). Wait for my reply "approved".

**Phase 3: Apply.** Carry out the approved plan exactly.

**Phase 4: Validate and report.** Check each item and fix any failure:
- [ ] No `.github/chatmodes/` folder and no `*.chatmode.md` file anywhere
- [ ] Every file has the right suffix, is in the right folder, and is named in kebab-case
- [ ] All frontmatter is valid YAML: required keys present, no disallowed keys, strings double-quoted
- [ ] Every agent `name` equals its file name; every skill `name` equals its folder name
- [ ] Every prompt `agent:` value is `ask`, `agent`, `plan` or an existing agent's `name`
- [ ] Every `tools` entry is an allowed name
- [ ] Every `applyTo` glob matches at least one file, or is listed as a question
- [ ] Size limits hold: 4,000 chars for instruction files, 30,000 chars for agents, 500 lines for SKILL.md
- [ ] Every relative link resolves
- [ ] No duplicated rules, no contradictions, no VS Code-only syntax

Final report: the new in-scope `.github` tree, a change table (before → after → why), every key and value you removed (including `model` values), and open questions.
