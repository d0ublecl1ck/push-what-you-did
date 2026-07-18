# Push What You Did

A reusable agent skill that commits and pushes only the files the agent edited for the current task.

## What it does

- Detects whether the workspace is a single repository or a multi-repo workspace
- Runs the same preflight checks as `commit-and-push`: status, diffs, log, remotes, upstream
- Builds a scoped owned-file list from files the agent directly modified
- Stages only owned paths instead of staging the entire worktree
- Groups owned paths into focused Conventional Commits (`type(scope): subject`)
- Guards secrets and local junk with the same rules as `commit-and-push`
- Re-stages hook changes only when they stay inside the owned path list
- Synchronizes with `git pull --ff-only` and asks before any rebase when diverged
- Pushes automatically after successful scoped commits

## How it differs from Commit&Push

The only difference is commit scope; everything else — safety guards, commit format, sync strategy — is identical.

- `commit-and-push` commits all worktree changes, including changes outside the current conversation
- `push-what-you-did` commits only the files the agent changed during the current task
- `push-what-you-did` stops if hooks or follow-up changes try to pull unrelated files into the commit

## When to use

Use this skill when you want your agent to commit and push only its own changes in a dirty worktree without touching unrelated edits. Use `commit-and-push` when every worktree change should be committed.

## How to trigger

Use natural language such as:

- `push what you did`
- `commit only your changes and push`
- `只提交并推送你这次改的文件`
- `不要碰别人的改动，把你做的提交上去`

Mentioning `$push-what-you-did` explicitly also triggers this skill.

## Installation

Place this folder under your agent skills directory, for example:

```bash
cp -R push-what-you-did <your-agent-skills-dir>/
```

## Safety notes

- Never stages all files with `git add -A`
- Never silently expands the owned file list
- Never commits unrelated untracked files just because they exist
- Never modifies git config
- Never uses interactive git commands
- Never force pushes implicitly; asks before any rebase when the branch has diverged
