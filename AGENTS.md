# openclaw-tools

OpenClaw integration tools and utilities.

---

## Screenshot Storage

Save every screenshot to `~/Desktop/screenshots/openclaw-tools/`.

```bash
export PROJECT_SCREENSHOT_DIR=~/Desktop/screenshots/$(basename "$(git rev-parse --show-toplevel)")
mkdir -p "$PROJECT_SCREENSHOT_DIR"
```

Do not keep proof or review screenshots in `/tmp`.

---

## Project Management — Keel

This project is managed by **keel** ([vykeai/keel](https://github.com/vykeai/keel)). Keel is the single source of truth
for tasks, specs, decisions, and roadmap. **Do NOT create or maintain manual TASKS.md,
ROADMAP.md, or similar tracking files.**

### With MCP access (preferred)
- Read state: `keel_status`, `keel_list_tasks`
- Start work: `keel_update_task { id, status: "active", assignee: "claude" }`
- Finish: `keel_update_task { status: "done" }` + `keel_add_note` with summary
- Blocked: `keel_update_task { status: "blocked" }` + `keel_add_note` with reason
- Architecture changes: `keel_update_architecture_doc`
- Decisions: `keel_log_decision` before implementing
- Search first: `keel_search "topic"` — update existing, don't duplicate

### Without MCP
- CLI: `keel status`, `keel tasks`, `keel task update <id> --status active`
- Read `views/` for current state — never edit views (generated)

---

## Execution — Cloudy

Use **cloudy** ([vykeai/cloudy](https://github.com/vykeai/cloudy)) for multi-task orchestration.

```bash
cloudy plan --spec ./docs/spec.md          # decompose spec → task graph
cloudy run --execution-model sonnet        # execute all tasks
cloudy check                               # re-validate completed tasks
```

**From inside Claude Code sessions** — unset nesting vars:
```bash
env -u CLAUDECODE -u CLAUDE_CODE_ENTRYPOINT cloudy run --spec spec.md
```

---

## Skills — Runecode

**runecode** ([vykeai/runecode](https://github.com/vykeai/runecode)) provides reusable Claude Code skills:

| Skill | Purpose |
|-------|---------|
| `/test-write` | Write tests for changed code |
| `/review-self` | Review your own code before committing |
| `/security-audit` | Audit changes for vulnerabilities |
| `/dead-code` | Find unused exports and unreachable code |
| `/tech-debt` | Identify technical debt |
| `/pr-description` | Write PR description from current diff |

**Project health**: `runecode doctor` checks setup. `runecode audit` scores and auto-fixes gaps.

---

## Package Conventions

- **Exports**: all public API must be declared in `package.json` `"exports"` field
- **"default" fallback**: every exports entry MUST have a `"default"` condition — webpack uses `require` on server-side and will throw "Package path not exported" without it
- **Types**: ship `.d.ts` declarations alongside JS output
- **ESM first**: prefer ESM (`"type": "module"`), provide CJS fallback via `"default"` condition where needed
- **Entry point**: `src/index.ts` re-exports public API — keep it thin (re-exports only, not a monolith)
- **Bundle config**: prefer `bundle: false` (each src file compiles to its own dist file) unless there's a specific reason to bundle

---

## Build

```bash
npm run build       # or: bun run build, tsup, tsc
npm run build:types # emit TypeScript declarations if separate
```

- `dist/` is generated — never hand-edit files in dist/
- After changes, rebuild and verify the package works when consumed by downstream projects
- If linked via yalc: `yalc push` in this repo, `yalc update` in consumers

---

## TypeScript — Zero Errors Policy

```bash
npx tsc --noEmit   # must output nothing
```

1. Never use `// @ts-ignore` or `// @ts-expect-error`
2. Never use `as any` — use `as unknown as TargetType` for double-cast
3. Prefer explicit types over `any`. Use `unknown` when truly unknown
4. All public API must have explicit type annotations

---

## Versioning

- Follow semver strictly
- Breaking changes = major version bump
- New features = minor version bump
- Bug fixes = patch version bump
- Document breaking changes in changelog or commit message

---

## Conventions

- Public types and shapes live in dedicated type files — keep compatibility intentional
- Keep packages independent — no circular imports between packages
- If this is a monorepo, each package must be independently buildable and testable
- Component libraries: use `"use client"` banner only on files that need browser APIs

---

## Testing

```bash
npm test        # or: bun test, vitest
```

- Test all public API exports
- Test type compatibility (consumer can import and use without errors)
- Test edge cases in utility functions

---

## Git Conventions

- Commit after every meaningful chunk of work
- Concise messages: `feat:`, `fix:`, `refactor:`
- Never commit `.env`, credentials, or dist/ artifacts

---

## Do Not

- Edit generated output in `dist/`
- Break public API without a major version bump
- Add `"use client"` to files that don't need it
- Import from dist/ paths — always import from src/ during development
- Create circular dependencies between packages
- Skip rebuilding after changes — downstream consumers will get stale code

---

## Definition of Done

- [ ] `tsc --noEmit` passes with zero errors
- [ ] `npm run build` succeeds
- [ ] All tests pass
- [ ] Exports are correct and consumable by downstream projects
- [ ] `/review-self` passed — no obvious issues in diff
- [ ] Changes committed (frequent, progressive — not batched at end)
- [ ] Keel task updated: `keel_update_task { status: "done" }` + `keel_add_note`

---

## Git Branch & Worktree Lifecycle

This section is mandatory for any agent that creates branches or worktrees.

### 1. Discover the gitflow before branching

Before creating any branch or worktree, determine the repo's integration model:

```bash
git remote show origin | grep "HEAD branch"   # default branch (usually main)
git branch -r | grep develop                  # does a develop branch exist?
```

- If `develop` exists → feature branches target `develop`, not `main`
- If only `main` exists → feature branches target `main`
- Never guess. Always check.

### 2. Worktree placement

**Never** place a worktree in `/tmp` or `/private/tmp`. Those paths are wiped on reboot, leaving orphaned `.git/worktrees/` metadata with no working directory.

Use one of these instead:
- `<repo-root>/.worktrees/<branch-name>/` — co-located, automatically gitignored
- `<repo-parent>/<repo-name>-<task-id>/` — sibling directory

### 3. Complete the full merge cycle before declaring done

A task is **not done** when code is written. It is done when:

1. All commits are on the feature branch
2. Feature branch is merged (via PR or `git merge`) into the target integration branch (`develop` or `main`)
3. Feature branch is deleted: `git branch -d <branch> && git push origin --delete <branch>`
4. Worktree is removed: `git worktree remove <path>`
5. Keel task is updated to `done`

Never leave a feature branch open after work is merged.

### 4. Abandoned or blocked tasks

If a task is blocked or abandoned mid-flight:
- Keep the branch on the remote (don't delete — work may resume)
- **Remove the local worktree**: `git worktree remove --force <path>`
- Update Keel status to `blocked` with a note linking the branch name

### 5. Audit before starting new work

Before opening a new worktree, run:

```bash
git worktree list
```

If you see more than 2–3 open worktrees, stop and clean up stale ones first. Accumulating worktrees causes merge conflicts, confusion, and disk waste.

### 6. Temporary/dirt branches

`temp-dirt-*` branches created as safety snapshots must be deleted within the same session once the real branch is verified. Never push `temp-dirt-*` branches to the remote as a permanent artifact.
