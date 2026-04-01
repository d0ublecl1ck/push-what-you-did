---
name: push-what-you-did
description: Strict workflow and safety constraints for committing and pushing only the files the agent modified in the current task or session, instead of all repo changes. Use when the user wants a scoped git commit/push, wants to avoid touching unrelated edits in a dirty worktree, or asks to "push what you did", "只提交你改的文件", or similar selective commit requests.
---

# Push What You Did

Use this skill when the user wants a git commit and push that includes only the files you edited for the current task.

## Workflow

1) **Do not widen scope**
- Do not stage all changes
- Do not touch user-authored or unrelated modified files
- Commit only the paths you directly created, edited, renamed, or deleted during the current task

2) **Detect repository shape first**
- Before any git action, determine whether the workspace is a single Git repository or a multi-repo workspace
- If multiple independent repositories are present, run this workflow separately inside each affected repository
- Skip repositories where you did not modify any files

3) **Build the owned file list**
- Derive the candidate path list from the files you edited in this task
- Include created files, modified files, renamed files, and deleted files that you directly touched
- Exclude files changed only by the user, tooling, formatters, generators, or hooks unless those changes are a direct consequence of your edits and stay within the same owned paths
- If you cannot determine the owned path list with high confidence, stop and ask instead of falling back to `git add -A`

4) **Run pre-checks in parallel**
- Run in parallel inside each affected repository: `git status`, `git diff`, `git log --oneline -5`
- Summarize the repository state before staging
- If the branch is both ahead and behind, the worktree is clean, and there are no owned pending paths, treat it as sync-only: run `git pull --rebase` and then `git push`

5) **Verify owned paths before staging**
- Check each owned path with `git status --short -- <path>` or equivalent
- Drop paths that no longer have changes
- If no owned paths remain after verification, do not create an empty commit

6) **Stage only owned paths**
- Never use `git add -A`, `git commit -a`, or any equivalent all-files shortcut
- Stage only the verified owned paths
- Use path-scoped staging that handles creates, edits, and deletions correctly
- Keep quoting safe for spaces and special characters in file names

7) **Commit from the staged diff**
- Write the commit message with HEREDOC
- Base the summary on `git diff --cached`
- Describe only the staged changes, not the full dirty worktree
- Never include signature lines or co-author trailers unless the user explicitly asks
- Do not create empty commits

8) **Handle hook rewrites carefully**
- If pre-commit hooks modify files, inspect which paths changed
- If hook changes stay entirely within the owned path list, re-stage only those owned paths and retry the commit
- If hooks modify files outside the owned path list, stop and ask the user instead of absorbing unrelated files into the commit

9) **Push after successful commit**
- Push only after the scoped commit succeeds
- If no upstream is configured, push with `-u origin <current-branch>`
- If push fails because the remote moved ahead, prefer `git pull --rebase` and retry
- If push still fails, treat it as an exceptional condition and ask the user

## Hard Rules

- Never update git config
- Never run interactive git commands
- Never silently expand the owned path list to make the commit succeed
- Never commit unrelated untracked files just because they are present
- Never convert this scoped workflow into the all-files behavior of `Commit&Push`

## Practical Staging Pattern

- For existing or new files, prefer path-scoped `git add -- <paths>`
- For deleted or renamed tracked paths, use path-scoped update staging as needed so removals are captured without staging unrelated files
- Recompute the final staged diff with `git diff --cached --stat` before committing

## Trigger Examples

- `push what you did`
- `只提交并推送你这次改的文件`
- `不要碰别人的改动，把你做的提交上去`
- `commit only your changes and push`
