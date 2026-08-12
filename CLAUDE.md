# Protfolio

Static personal portfolio website. There is currently **no package manager, build
system, or automated test/lint/type-check tooling** in this repository — don't
assume `npm`/`yarn`/`pnpm` scripts exist; verify by checking for `package.json`
first.

## Structure

- `index.html` — the site itself (markup, styling, and any scripts)
- `screen1.jpg`, `screen2.png`, `screen3.png` — screenshots referenced from the README
- `demo.mp4` — demo video
- `README.md`

## Conventions

- Branches follow `<issue-number>-<kebab-case-slug-of-issue-title>`, matching the
  linked GitHub issue (e.g. `1-fix-portfolio-navbar-bug`).
- Default branch: `main`.
- No test/lint/build commands exist yet. If a task needs one, say so explicitly
  rather than inventing an `npm run ...` that isn't defined anywhere.

## Automation

- `/workflow` (`.claude/commands/workflow.md`) is a multi-agent GitHub
  issue-to-PR orchestrator: `/workflow` (interactive), `/workflow issue <n>`,
  `/workflow bugs`, `/workflow all`. It discovers issues, lets you pick
  one/many/all, and drives 9 specialized subagents
  (`.claude/agents/*.md`) through analysis, planning, implementation, testing,
  debugging, review, commit/push/PR, issue linking, and (when configured)
  GitHub Project status + preview deployment — pausing for your explicit
  approval before starting an issue, before commit+push, before opening a PR,
  and before any deployment trigger.
- Tunable behavior (retry budgets, protected branches, discovery labels) lives
  in `.claude/workflow/config.json`, not hardcoded in the command.
- Full design rationale: `.claude/workflow/ARCHITECTURE.md`.
- Per-run state/logs land in `.claude/workflow/state/` and
  `.claude/workflow/logs/` (both gitignored — local run history, not project
  content).
