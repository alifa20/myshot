---
name: cmux-tab-git-worktree
description: Create one or more git worktrees under ../<repo>-worktrees/ and open a cmux terminal tab already cd'd into each. Use when the user asks to spin up a worktree, run branches in parallel, or wants a cmux tab per branch.
---

# cmux tab per git worktree

Creates a git worktree (new branch) in a sibling `../<repo>-worktrees/<name>` folder and opens a
named cmux terminal tab whose shell starts in that folder. Repeat per branch to run several
lines of work in parallel.

## Inputs

- **name** — worktree folder name and tab title (kebab-case).
- **branch** — branch to create. Default `feat/<name>`.
- **base** — commit/branch to branch from. Default current `HEAD`.

If the user names source docs or tasks instead of branch names (e.g. "one for each of these two
prompts"), derive one `name` per item and confirm nothing else is needed.

## Steps

1. **Check the tree is committed.** Worktrees branch from a commit, so anything staged/untracked
   that the new branch needs must be committed first:

   ```bash
   git status --short
   ```

   If needed files are uncommitted, commit them on the current branch *before* creating worktrees
   and tell the user you did. Don't silently leave the worktree missing them.

2. **Create the worktree(s).** Resolve the repo root and name:

   ```bash
   ROOT=$(git rev-parse --show-toplevel)
   REPO=$(basename "$ROOT")
   git worktree add "$ROOT/../$REPO-worktrees/<name>" -b <branch> [<base>]
   ```

3. **Open a cmux tab per worktree**, in the *current* workspace, with the shell already there.
   Use an absolute path — `--working-directory` does not expand `~` or relative paths reliably:

   ```bash
   cmux new-surface --type terminal \
     --working-directory "$ROOT/../$REPO-worktrees/<name>" \
     --focus false
   ```

   It prints `OK surface:<N> pane:<P> workspace:<W>`. Capture `surface:<N>`.

4. **Name the tab** so the branches are tellable apart:

   ```bash
   cmux rename-tab --surface surface:<N> "<name>"
   ```

5. **Report** a small table: worktree path, branch, tab ref.

## Notes

- Tabs vs workspaces: `cmux new-surface` makes a **tab** in the current workspace — that is what
  this skill wants. `cmux workspace create` (aka `new-workspace`) makes a whole new sidebar
  **workspace**; only use it if the user explicitly asks for a separate workspace.
- Keep `--focus false` so the session driving the setup isn't yanked away. The user can switch
  with `cmux tab-action --tab tab:<N> --action ...` or just click.
- Want an agent already running in the tab instead of a bare shell? Use
  `cmux new-surface --type agent-session --provider claude --working-directory <path>`.
- Cleanup, when the user asks: `git worktree remove <path>` then `git branch -d <branch>`;
  close the tab with `cmux close-surface --surface surface:<N>`.
- `cmux --help` and `cmux docs api` are authoritative if a flag here looks stale.
