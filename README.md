# ship

A [Claude Code](https://code.claude.com) skill that does the git work at the end of a task, so
you stop typing the same five commands.

- **`/ship`** branches off the default branch if you are on it, commits, pushes and opens a pull
  request.
- **`/ship merge`** does all of that, then squash-merges the PR, deletes the branch, pulls the
  default branch and prunes stale local branches. `merge` has to be the first word.

Anything else you type after `/ship` is guidance for the branch name, commit message and PR
description.

## What it checks before it acts

- **It pins every `gh` command to `origin`.** In a clone with an `upstream` remote, `gh` on its
  own resolves to the upstream repo, which means someone else's project. Every PR command passes
  `-R <origin url>`.
- **It branches first.** Work on the default branch moves to `feat/`, `fix/`, `docs/` or `chore/`
  before the commit. Unpushed commits on the default branch come along and are named in the
  report. The exception is a repo with no remote, or a remote with no branches yet: there it
  commits where you are.
- **It looks for secrets.** Staged files, and every commit about to be pushed, are searched for
  common key, token, password and connection-string shapes, and the matching lines are read. It
  is a backstop, not a guarantee: read your diff before shipping to a public repo.
- **It holds back files that rarely belong in git.** `.env` files, keys, embedded repos, files
  over 10 MB, the first data file of a new type (`.csv`, `.xlsx`, `.parquet` and similar), opaque
  binaries, and Power BI local state (`.pbi/cache.abf`, `.pbi/localSettings.json`, a new `.pbix`)
  are unstaged and reported with the `.gitignore` line that would keep them out.
- **It stops instead of forcing.** A rejected push, a conflict, a failing hook, a detached HEAD,
  a long-lived branch such as `develop`, or a switch that would overwrite local files ends the run
  with a report. It never force-pushes, never merges with `--admin`, and only force-deletes a
  local branch when that branch's tip is the head of a merged PR.
- **It guards Power BI Desktop.** In a PBIP repo with [pbir](https://github.com/maxanatsko/pbir.tools)
  installed and Desktop's local-API preview on, it checks whether Desktop has the report open with
  unsaved changes. If it does, the files on disk are stale and it asks you to save first.
- **It only merges when you say `merge`.** Plain `/ship` stops at an open PR.

## Requirements

- Claude Code with skills support.
- git 2.42 or later. On Windows, Git for Windows, because the skill runs its commands through
  Claude Code's Bash tool (Git Bash).
- The [GitHub CLI](https://cli.github.com) 2.57 or later, authenticated, for pull requests.
  Without it, or with a remote that is not on GitHub, `/ship` commits and pushes and stops there.
- Optional: the pbir CLI for the Desktop guard. `pbir desktop list` needs Power BI Desktop's
  "Enable external tool access to Power BI Desktop through secure local APIs" preview turned on.

## Install

Clone it into your personal skills folder:

```bash
git clone https://github.com/InsightfulAnalytics/ship ~/.claude/skills/ship
```

On Windows PowerShell:

```powershell
git clone https://github.com/InsightfulAnalytics/ship "$env:USERPROFILE\.claude\skills\ship"
```

Start a new Claude Code session and type `/ship`. Update later by running `git pull` in that
folder.

The skill sets `disable-model-invocation: true`, so Claude never runs it on its own and it adds
nothing to your context until you type it.

## Optional: commit after every task

Claude Code's default is to commit only when asked. To have it commit as it goes and leave the
push and PR to `/ship`, add this to your `~/.claude/CLAUDE.md`:

```markdown
## Git: commit without being asked

This is a standing request to commit (the ship skill's auto-commit rule), so it replaces the
default of committing only when asked.

- Before the first edit of a task in a git repo, note `git status --short`: the paths it lists
  are the user's work in progress and stay unstaged.
- When the main session finishes a task that changed files in a git repo, commit it by following
  steps 1 to 4 of `~/.claude/skills/ship/SKILL.md` (read it first). Subagents leave committing to
  the session that dispatched them.
- Pushing, PRs and merging wait for `/ship` or `/ship merge`.
- A repo's own CLAUDE.md can override this: "commit on the current branch" skips step 3 there,
  and "no auto-commit" means no commits at all.
```

Under that rule, Claude stages only the files the task changed (plus a new `.gitignore` when the
repo has none) and leaves your own work in progress alone. The skill's pre-approved commands
apply only when you type `/ship`, so expect permission prompts for the git and gh commands in
steps 1 to 4 (fetch, switch, add, commit and others) unless your settings allow them.

## Limits

- One Claude session per checkout. `/ship` stages the whole tree and switches branches, so a
  second session's unfinished edits would ride in the PR. Give parallel sessions their own
  worktree.
- Pull requests open on `origin`. If `origin` is your fork and you want a PR against the upstream
  project, open that one yourself.
- Pull requests need GitHub. GitLab, Azure DevOps and other hosts get commit and push only.
- `/ship merge` does not wait for CI: only checks that branch protection or a ruleset requires
  hold the merge.
- Squash merge is the default. When a repo refuses squash merges, it falls back to a merge commit,
  then a rebase, and says which.
- With a merge queue, `/ship merge` queues the PR and stops. Run `/ship merge` again once it lands
  to pull and prune, and delete the remote branch yourself unless the repo auto-deletes head
  branches.
- Built on Windows 11 with git 2.55, gh 2.96 and Claude Code 2.1.

## License

[MIT](LICENSE)
