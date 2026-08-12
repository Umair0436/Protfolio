# `/workflow` Architecture

Living reference for how the `/workflow` command is built. Keep this in sync
whenever `.claude/commands/workflow.md` or `.claude/agents/*.md` change.

## Mechanism

`/workflow` is a **Claude Code custom command**
(`.claude/commands/workflow.md`), not a skill or a single subagent. It has to
run in the main conversation because it pauses for real user approval at
several points — a backgrounded skill/subagent can't do that. It acts as the
**Orchestrator**: it holds workflow state and sequencing, but delegates every
specialized step to one of 9 custom subagents (`.claude/agents/*.md`) via the
`Agent` tool.

## Why subagents, and why 9 of them

Each subagent is a separate Claude Code agent *type*, defined by its own
frontmatter (`name`, `description`, `tools`). Splitting the pipeline this way
gives two real benefits, not just organizational tidiness:

1. **Tool scoping is the actual safety mechanism.** The Orchestrator itself has
   no `git commit`, `git push`, `gh pr create`, or GitHub-write tool in its own
   `allowed-tools`. Only `git-pr-agent` can commit/push/open a PR, only
   `issue-manager-agent` can write to GitHub Issues/Projects, and neither has a
   merge or delete tool anywhere in this system. A prompt-injection attempt
   embedded in an issue body, or a reasoning mistake by the Orchestrator, still
   can't reach a dangerous operation without going through an agent whose tool
   list physically doesn't include it.
2. **Each agent is invoked with a fresh, minimal, self-contained context.**
   Subagents don't share the Orchestrator's conversation — they only see what's
   in the prompt they're given. This keeps e.g. the Testing Agent from being
   influenced by the Planning Agent's reasoning, and keeps failed debug attempts
   from polluting the Code Review Agent's judgment.

| Agent | Tools it has | Tools it deliberately lacks |
|---|---|---|
| `issue-manager-agent` | `gh issue`/`gh project` read+comment | close, delete, merge, any git tool |
| `repo-analysis-agent` | Read/Grep/Glob, read-only git | Edit, Write, any write tool |
| `planning-agent` | Read/Grep/Glob | Edit, Write, any git/gh tool |
| `coding-agent` | Read/Edit/Write, read-only git | commit, push, gh |
| `testing-agent` | Read, scoped run-only Bash | install/ci, git-write, gh |
| `debugging-agent` | Read/Edit/Grep/Glob, read-only git | commit, push, gh |
| `code-review-agent` | read-only git, Read/Grep | Edit, Write, git-write, gh |
| `git-pr-agent` | commit, push (non-protected branches only), `gh pr create` | merge, force-push, delete, project-write |
| `deployment-agent` | Read/Grep/Glob, `gh workflow` | git-write, `gh pr`, delete |

## State and logging

- `.claude/workflow/config.json` — tunables: `maxDebugAttempts`,
  `maxReviewRevisionAttempts`, `protectedBranches`, `issueDiscoveryLabels`,
  `defaultBranch`. This is what "configurable number of attempts" (per the
  safety rules) actually refers to — edit this file to change the budgets, no
  need to edit the command itself.
- `.claude/workflow/state/<issue-number>.json` — current phase, branch, attempt
  counts, `status` (`IN_PROGRESS` / `BLOCKED` / `COMPLETED` / `SKIPPED`),
  `blockedReason`, `prUrl`. Gitignored — this is local run state, not project
  content.
- `.claude/workflow/logs/<issue-number>.md` — append-only, timestamped record of
  every phase and which agent handled it. Gitignored for the same reason.

## Approval gates

Four distinct points, each its own yes/no — approving one never implies
approval for the next:

1. **Start this issue** (before branch/analysis/planning/coding begin).
2. **Commit + push** (after tests pass and review approves).
3. **Open a PR** (separate from commit+push, even though it usually follows
   immediately).
4. **Trigger a deployment** (only asked if one is actually configured).

`all` mode collapses the *selection* question (which issues) into one batch
confirmation, but never collapses gates 1–4 for the individual issues in that
batch — each issue still gets its own commit/PR/deploy approvals.

## BLOCKED state

If the debug loop (step 8) or the review-revision loop (step 9) in the per-issue
pipeline exhausts its configured attempt budget without success, the issue's
state file is set to `BLOCKED` with a `blockedReason`, the pipeline stops for
that issue specifically (no commit, no PR), and the Orchestrator moves on to the
next issue in a batch, or ends the run if it was the only one selected. Blocked
issues are called out explicitly in the final summary.

## Known gaps / future extensions

- This repo has **no test/lint/build tooling** today — `testing-agent` will
  report that plainly rather than inventing commands; step 7/8 fall back to a
  manual diff read when that's the case.
- This repo has **no deployment configuration** today — `deployment-agent` will
  report "not configured" and skip step 14.
- Issues are processed **sequentially in one working tree** (branch-switch, not
  parallel worktrees). True concurrent multi-issue processing would need
  per-issue git worktrees — a deliberate future upgrade, not built here.
- `gh` CLI is not installed/authenticated on this machine as of this writing —
  Setup step 2 in the command will detect and stop on this rather than fail
  partway through.
