---
name: atomize
description: >
  Breaks down code changes into atomic commits in two modes. Review mode: replay a branch's
  diff as maximally atomic commits in a separate worktree for careful review. Commit mode:
  atomize unstaged/uncommitted changes into clean logical commits on the current branch.
  Use when the user wants to review a branch, split changes, or atomize working changes.
  Trigger on "review this branch", "split these commits", "atomize", "break this down", or
  similar requests. Ask the user which mode if not specified. Commits are split within files
  when logically distinct changes exist.
---

# Atomize

Breaks down code changes into atomic commits in two modes. Review mode: replay a branch's
commits as a series of logical, single-purpose commits in an isolated worktree for careful
inspection using `git log` and `git show`. Commit mode: take unstaged/uncommitted changes
in the working directory and atomize them into clean logical commits on the current branch.
Large diffs and tangled working directories are hard to reason about; this skill breaks them
down into comprehensible units.

## Step 1: Determine the mode

If the user hasn't specified, ask which mode they want:

- **Review mode**: Break a branch's existing commits into atomic commits in an isolated worktree
  for careful review. You review at your own pace and signal when done; the worktree is then
  cleaned up. Original branch stays unchanged.
- **Commit mode**: Take unstaged/uncommitted changes in the current working directory and atomize
  them into a series of clean, logical commits on the current branch.

## Step 2: Assess changes

1. Confirm the repository root: `git rev-parse --show-toplevel`
2. Identify the current branch: `git branch --show-current`

**For Review Mode:**
3. Identify the base branch (prefer `main`, then `master`, then ask the user if neither exists):
   ```
   git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
   ```
4. Get the full diff between the merge base and HEAD:
   ```
   git diff <merge-base>..HEAD
   ```

**For Commit Mode:**
3. Check for unstaged/uncommitted changes:
   ```
   git status
   git diff
   ```
   If no changes exist, inform the user there's nothing to atomize.

## Step 3: Create the worktree (Review Mode only)

Choose a path next to the repo root: `<repo-parent>/<repo-name>-atomize`

```bash
git worktree add <worktree-path> <merge-base>
```

This creates a clean checkout at the merge base — the exact point where the branch diverged.

## Step 4: Plan the atomic commits

Analyze the diff and group changes into the smallest logical units that still make sense
in isolation. Good rules of thumb:

- Each commit should have a single, clearly statable purpose
- Changes to different concerns in the same file belong in different commits (e.g. adding
  a new method vs. fixing a bug in an existing one)
- Related changes across multiple files that serve one purpose belong in the same commit
  (e.g. adding a function and its test)
- Refactors and functional changes should never be in the same commit
- Imports/dependencies added to support a change belong with that change

Think in terms of what someone would write in a commit message. If a reasonable message
would use "and" to describe two separate things, it should be two commits.

Create a mental (or written) ordered list of commits before you start applying any of
them. More commits is better — err on the side of splitting.

## Step 5: Apply commits in the worktree

For each planned commit, working inside the worktree directory:

1. Apply only the relevant changes using `git apply --index` with a patch, or by directly
   editing files and staging with `git add -p` for partial-file staging
2. Write a clear, concise commit message (imperative mood, present tense)
3. Commit: `git commit -m "<message>"`

Apply commits in a logical order — typically: infrastructure/dependencies first, then
core logic, then tests, then docs/style.

**Partial file staging tip:** When a file has multiple distinct changes, use patch-mode
to stage only the relevant hunks:
```bash
git -C <worktree-path> add -p <file>
```

## Step 6: Mode-specific finalization

### Review Mode

Tell the user atomic commits are ready for review:

```
Atomic commits are ready for review at: <worktree-path>

There are <N> commits. You can review them with:
  cd <worktree-path>
  git log --oneline          # see all commits
  git show <hash>            # inspect a specific commit
  git log -p                 # see all diffs inline

Let me know when you're done reviewing and I'll clean everything up.
```

Wait for the user to signal they're done reviewing (e.g., "done", "finished", "cleanup").

### Commit Mode

Stash the current changes and begin applying them as atomic commits:

```bash
git stash
```

Then, for each planned commit:
1. Apply only the relevant changes by editing files directly or using patch mode
2. Stage the changes: `git add <files>` or `git add -p <file>` for partial staging
3. Write a clear commit message (imperative mood, present tense)
4. Commit: `git commit -m "<message>"`

Apply commits in a logical order — typically: infrastructure/dependencies first, then core
logic, then tests, then docs/style.

After all atomic commits are applied, tell the user:

```
Atomized commits created! <N> new commits have been added to your branch.

You can review them with:
  git log --oneline          # see all commits
  git log -p                 # see all diffs inline

All your changes are now organized into clean, logical commits.
```

## Step 7: Cleanup (Review Mode only)

Remove the worktree and prune stale refs:

```bash
git worktree remove --force <worktree-path>
git worktree prune
```

Confirm: "Worktree removed. All clean."

