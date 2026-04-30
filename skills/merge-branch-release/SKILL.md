---
name: merge-branch-release
description: Use when the user wants to merge the current branch into another branch, publish code, push to a target branch, release to test/pre-release/prod, or says to merge and push.
---

# Merge Branch Release

## Core Rule

Preserve the user's working branch. Commit current-branch work first, merge that
branch into the target branch, push the target branch, then switch back to the
original branch.

## Worktree Target Resolution

Before switching branches, detect worktree state and resolve the target branch
in this order:

1. Explicit user target branch.
2. Explicit release lane alias mapped to an existing branch, for example `main`, `master`, `develop`, `test`, `pre`, or `prod`.
3. Remote default branch from `git symbolic-ref --short refs/remotes/origin/HEAD`, if the user asked for the trunk but did not name it.
4. Local trunk fallback: `main`, then `master`, then `develop`.
5. Other checked-out worktree branch, only when the user clearly requested that branch.

When the target branch is checked out in another worktree:

- Do not force checkout that branch in the current worktree.
- If the branch cannot be checked out because Git reports it is already checked out elsewhere, use the path from `git worktree list --porcelain`.
- Before running target steps there, run `git -C <target_worktree_path> status --short`; stop if that worktree has unrelated uncommitted changes.
- Commit source changes first, then run the target-branch update, merge, push, and verification commands from that target worktree path.
- Switch the user's current shell back to the original `repo_root` after verification.

## Workflow

1. Capture starting state:
   - `source_branch=$(git branch --show-current)`
   - `repo_root=$(git rev-parse --show-toplevel)`
   - `git_common_dir=$(git rev-parse --git-common-dir)`
   - `git worktree list --porcelain`
   - `git status --short`
   - `git diff --staged; git diff`
   - `git log --oneline -10`

2. Determine whether the current checkout is part of a Git worktree setup.
   - Parse `git worktree list --porcelain`.
   - Match the current `repo_root` to a `worktree <path>` entry.
   - Treat the checkout as a worktree checkout when the repository has more than one `worktree` entry or the matched path is not the primary worktree path.
   - Record all checked-out branches from `branch refs/heads/<name>` entries.

3. Determine the target branch from the user request.
   - If unclear, ask.
   - If the target branch is the same as `source_branch`, stop and clarify.
   - If the user explicitly names a target branch, use that branch.
   - If the user says "main", "master", "trunk", "主干", "test", "pre", "prod", or another release lane, resolve it to the matching local or remote branch.
   - If the current checkout is a worktree and the user asks to merge to the trunk, merge into the trunk branch.
   - If the current checkout is a worktree and the user asks to merge to another branch that is checked out in a different worktree, merge into that requested branch.
   - Do not infer the target from worktree paths alone when the user named a branch; the named branch wins.

4. Commit current branch code before switching.
   - Review staged and unstaged changes.
   - Stage only relevant files for the requested work.
   - Do not commit secrets or unrelated user changes without explicit confirmation.
   - Commit with a concise message via heredoc:

```bash
git commit -m "$(cat <<'EOF'
feat(scope): concise intent

EOF
)"
```

5. Check whether the target branch exists.
   - Fetch branch refs first when network is needed: `git fetch origin`
   - If local target exists: `git checkout <target>`
   - Else if remote target exists: `git checkout -b <target> origin/<target>`
   - Else create target from current branch state: `git checkout -b <target> <source_branch>`
   - If the target branch is checked out in another worktree, do not run `git checkout <target>` here; run target-branch steps inside that worktree path.

6. Update the target branch if it tracks a remote.
   - If target steps run in another worktree path, first verify that path is on `<target>` and has no unrelated uncommitted changes.
   - If target has upstream: `git pull --ff-only`
   - If pull cannot fast-forward, stop and report; do not rebase or reset unless explicitly requested.

7. Merge source into target.
   - If target was newly created from source, no merge is needed.
   - Otherwise run: `git merge --no-ff <source_branch>`
   - If conflicts occur, stop after reporting conflicted files unless the fix is obvious and requested.

8. Push target.
   - Existing upstream: `git push`
   - No upstream or newly created branch: `git push -u origin <target>`

9. Verify and switch back.
   - Run `git status -sb` and `git log --oneline -3`.
   - Switch back: `git checkout <source_branch>`.
   - Run `git status -sb` again and tell the user whether source is ahead/behind its remote.
   - If target steps ran in another worktree path, return to the original `repo_root` instead of switching that target worktree away from its branch.

## Safety Constraints

- Never use `git reset --hard`, `git checkout --`, force push, or rebase unless the user explicitly requests it.
- Never skip hooks with `--no-verify`.
- Do not push the source branch unless the user asks; only push the target branch for this workflow.
- Do not remove, prune, or modify worktrees unless the user explicitly requests it.
- If uncommitted unrelated changes are present, ask before including them.
- If a command needs network or unrestricted git access, request the needed tool permission and continue.

## Final Response

Report:

- commit hash created on the source branch, if any
- target branch updated
- whether a separate worktree path was used for the target branch
- push result
- final branch after switching back
- any residual ahead/behind status or conflicts
