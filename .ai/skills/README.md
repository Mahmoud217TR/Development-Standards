# AI Skills

This folder contains AI-optimized versions of the team standards. Each skill is a self-contained markdown file with rules, code patterns, and decision trees designed to be loaded into AI coding assistants.

## Audience

These files are written for **AI tools, not humans.** They are:

- Directive (imperative, not explanatory)
- Code-pattern-heavy (skeletons the AI can generate from)
- Free of rationale and "we considered alternatives" content
- Optimized for token efficiency relative to the full standards

Humans should read the corresponding numbered sections in the repo root (`04-architecture-patterns.md`, `09-testing-strategy.md`, etc.) — those have the rationale and discussion.

## Skills

| File | Source section | Covers |
|---|---|---|
| [`architecture.md`](./architecture.md) | Section 4 | Actions, Queries, Services, DTOs, Events, Listeners, Jobs, Controllers, FormRequests, Resources, Models, States, Exceptions — folder structure, naming, dependency rules |
| [`testing.md`](./testing.md) | Section 9 | Pest, Postgres + RefreshDatabase, feature vs unit tests, Http::fake / interface fakes, Listener / Job test patterns, factories, CI |

## How AI tools load these

Tool-specific pointer files at the repo root forward each AI to this folder:

- `CLAUDE.md` → Claude Code
- `AGENTS.md` → OpenCode, Sourcegraph Cody, Cline, Continue, and the emerging cross-tool standard
- `.github/copilot-instructions.md` → GitHub Copilot
- `.cursorrules` → Cursor

Each pointer is 3–10 lines and tells the tool to load and follow the skills in this folder.

## Adding a new skill

1. Distill the relevant numbered section(s) into AI-directive form
2. Include code skeletons for every component type covered
3. Include a forbidden-patterns list at the end
4. Include a generation checklist
5. Update this README's table
6. Update each pointer file's skill list

## Sync with main standards

Skills must stay in sync with their source sections. When a numbered section changes (e.g., Section 4 rule added), update the matching skill file in the same PR. Drift between the standards and the skills will produce inconsistent AI output and human/AI disagreement on what's correct.
