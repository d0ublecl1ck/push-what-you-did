---
name: push-what-you-did
description: Commit and push only the files the agent modified in the current task or session, instead of all worktree changes. Use when the user wants a scoped git commit/push, wants to avoid touching unrelated edits in a dirty worktree, or asks to "push what you did", "只提交你改的文件", or similar selective commit requests. Everything except commit scope mirrors commit-and-push; for committing all worktree changes, use commit-and-push instead.
---

# Push What You Did

Commit and push only the files you directly created, edited, renamed, or deleted during the current task. This skill mirrors `commit-and-push` in every respect — safety guards, commit format, sync strategy — except commit scope: `commit-and-push` commits all worktree changes, this skill commits only your own.

## Activation Boundary (hard rule)

- This skill is opt-in only.
- Never invoke this skill proactively, implicitly, by default, or because it seems helpful.
- Use it only when the user explicitly asks for a scoped commit and push of the agent's own changes.
- A request to commit all worktree changes activates `commit-and-push`, not this skill.

## Scope Boundary (the only difference from commit-and-push)

- Commit only the paths you directly created, edited, renamed, or deleted during the current task.
- Derive the candidate path list from the files you edited in this task.
- Exclude files changed only by the user, tooling, formatters, generators, or hooks unless those changes are a direct consequence of your edits and stay within the same owned paths.
- Never stage all changes, and never touch user-authored or unrelated modified files.
- If you cannot determine the owned path list with high confidence, stop and ask instead of falling back to `git add -A`.
- Verify each owned path with `git status --short -- <path>` before staging; drop paths that no longer have changes.
- If no owned paths remain after verification, do not create an empty commit.

## Safety Invariants

- Never change Git configuration outside disposable verification fixtures.
- Never use destructive commands such as `git reset --hard`, `git clean -fd`, or force push unless the user explicitly authorizes that exact action.
- Never use interactive Git commands such as `git rebase -i`.
- Never commit `.env`, credentials, private keys, tokens, cookies, `node_modules/`, `.venv/`, `__pycache__/`, or large binaries without explicit approval.
- Never overwrite, revert, or discard changes that were not produced by the current task.
- Never create an empty commit unless explicitly requested.
- Create a PR only when explicitly requested.

## Mandatory Workflow

### 1. Discover repository boundaries

Before any staging or commit:

1. Determine whether the current workspace is one repository, a worktree, or a directory containing multiple independent repositories.
2. Discover repositories from Git metadata; do not rely on machine-specific absolute paths or hard-coded project names.
3. Treat each repository independently for status checks, staging, hooks, synchronization, commits, and push outcomes.
4. Skip repositories where you did not modify any files.
5. If repository ownership is ambiguous, stop and ask which repositories are in scope.

### 2. Run preflight checks in parallel

For every affected repository, run these read-only checks in parallel:

```bash
git status --short --branch
git diff --stat && git diff
git diff --cached --stat && git diff --cached
git log --oneline -5
git remote -v
git branch --show-current
git rev-parse --abbrev-ref --symbolic-full-name '@{upstream}'
```

The upstream query may fail when no upstream exists; record that state instead of treating it as a fatal error.

Summarize before changing the index:

- repository and branch;
- owned pending paths versus other changes left untouched;
- upstream and ahead/behind state;
- likely commit groups within the owned paths;
- owned files blocked by safety rules.

### 3. Classify exceptional states

#### First commit

If `git log --oneline -5` reports no commits, inspect the owned files and create the initial commit from the owned paths without asking routine questions.

#### Sync-only

Treat a repository as sync-only only when all are true:

- the worktree and index are clean, or no owned path has pending changes;
- upstream is ahead and local has no unpublished commits;
- the repository already has commits.

Run `git fetch`, verify divergence with `git rev-list --left-right --count HEAD...@{upstream}`, then use `git pull --ff-only` and `git push`; do not create a new commit. If both sides have commits, classify the state as diverged rather than sync-only and ask before rebasing.

#### No owned changes

If no owned path has pending changes and the branch is already synchronized, report "nothing to commit or push" and stop without an empty commit.

#### Detached HEAD or unresolved conflicts

Stop and report the exact state. Do not invent a branch, resolve conflicts, or push from detached HEAD without user direction.

### 4. Build a commit plan from owned paths

Review the owned path list and group owned files into commit units.

- Group by coherent intent visible in the diff; one coherent intent is one commit unit.
- Keep unrelated owned changes separate even when they live in the same repository.
- Never pull non-owned files into a unit to make the grouping cleaner.
- Do not use `git add -A` across all groups.

Before staging, present a concise plan in the same format as `commit-and-push`:

```text
Commit 1 — feat(validation): add URL validation
  src/validation.ts
  tests/validation.test.ts

Left untouched (not owned)
  src/other.ts — modified outside this task
```

Proceed autonomously after the plan unless an exceptional condition or ambiguous file ownership requires a question.

### 5. Guard secrets and local junk

Inspect owned untracked and staged files before every commit unit.

Local-only junk includes `.DS_Store`, `*.pyc`, `__pycache__/`, editor swaps, temporary logs, coverage output, build caches, and local virtual environments.

- Add the narrowest appropriate pattern to a repository `.gitignore` when the rule should be shared.
- Use `.git/info/exclude` only for deliberately machine-local exclusions.
- If an owned junk file is already tracked, show the exact path, add its ignore rule, then use `git rm --cached -- <path>` and verify the local file still exists before committing. Never use plain `git rm` for this case.
- Commit shared ignore-rule changes with the commit unit that exposed the junk.
- Never auto-ignore source, assets, fixtures, migrations, lockfiles, or ownership-ambiguous files.

Secret-like files are blocked, not merely ignored. Stop and ask if the user explicitly wants one committed.

### 6. Stage and commit each unit

For each commit unit, in chronological order:

1. Stage only that unit's owned paths; use path-scoped staging that handles creates, edits, renames, and deletions correctly, with safe quoting for spaces and special characters.
2. Inspect `git diff --cached --stat` and `git diff --cached`.
3. Confirm no blocked file or non-owned hunk is staged.
4. Draft the message from the staged diff; describe only the staged changes, not the full dirty worktree.
5. Commit with a HEREDOC.

Commit message contract:

- Format: `type(scope): subject`.
- Scope is required and kebab-case.
- Subject is present-tense imperative, concrete, concise, and has no trailing period.
- Add a body for non-trivial changes to explain how and why.
- Use trailers only when they add traceability.
- Use `!` or `BREAKING CHANGE:` for breaking changes.
- Never add generated-by signatures or synthetic co-author lines.

```bash
git commit -m "$(cat <<'EOF'
fix(auth): use constant-time key comparison

Replace direct equality to avoid timing differences when checking API keys.
EOF
)"
```

If a hook modifies files, inspect which paths changed. If hook changes stay entirely within the owned path list, re-stage only those owned paths and retry the commit. If hooks modify files outside the owned path list, stop and ask the user instead of absorbing unrelated files into the commit.

### 7. Synchronize and push

After all commit units in a repository succeed:

1. Run `git fetch` and re-check the worktree, index, upstream, and exact divergence with `git rev-list --left-right --count HEAD...@{upstream}`.
2. If the worktree or index contains unexpected changes, stop before synchronization.
3. If upstream is ahead and local has no unpublished commits, run `git pull --ff-only`.
4. If both sides have commits, do not rewrite local history automatically. Report the divergence and ask before any rebase.
5. Push once to the tracked branch after synchronization is safe.
6. If no upstream exists, use `git push -u origin <current-branch>`.
7. Never force push implicitly.

A push failure is exceptional: report the command, error, committed local SHAs, and unpushed branch; then ask what to do.

### 8. Report outcomes

Summarize per repository:

- commit SHA and subject for each unit;
- pushed branch and remote;
- owned files skipped or blocked;
- non-owned changes deliberately left untouched;
- hook, rebase, or push exceptions.

## Hard Rules

- Never update git config.
- Never run interactive git commands.
- Never silently expand the owned path list to make the commit succeed.
- Never commit unrelated untracked files just because they are present.
- Never convert this scoped workflow into the all-files behavior of `commit-and-push`.

## Practical Staging Pattern

- For existing or new files, prefer path-scoped `git add -- <paths>`.
- For deleted or renamed tracked paths, use path-scoped update staging as needed so removals are captured without staging unrelated files.
- Recompute the final staged diff with `git diff --cached --stat` before committing.

## Trigger Examples

- `push what you did`
- `只提交并推送你这次改的文件`
- `不要碰别人的改动，把你做的提交上去`
- `commit only your changes and push`
