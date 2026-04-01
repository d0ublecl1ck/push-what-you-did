# Push What You Did

A reusable agent skill that commits and pushes only the files the agent edited for the current task.

## What it does

- Detects whether the workspace is a single repository or a multi-repo workspace
- Runs pre-checks before staging: `git status`, `git diff`, `git log --oneline -5`
- Builds a scoped owned-file list from files the agent directly modified
- Stages only owned paths instead of staging the entire worktree
- Writes commit messages from the staged diff via HEREDOC
- Re-stages hook changes only when they stay inside the owned path list
- Pushes automatically after a successful scoped commit

## How it differs from Commit&Push

- `Commit&Push` commits all current changes in the repository, including unrelated files
- `Push What You Did` commits only the files the agent changed during the current task
- `Push What You Did` stops if hooks or follow-up changes try to pull unrelated files into the commit

## When to use

Use this skill when you want your agent to commit and push only its own changes in a dirty worktree without touching unrelated edits.

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
