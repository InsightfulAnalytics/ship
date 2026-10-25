---
name: ship
description: "Ship the current work: branch off the default branch if needed, commit, push, open a PR. `/ship merge` also lands it once it is up to date, conflict-free and every check has passed, then prunes."
argument-hint: "[merge] [notes]"
disable-model-invocation: true
allowed-tools: Bash(git status *) Bash(git rev-parse *) Bash(git remote -v) Bash(git remote get-url *) Bash(git remote set-head *) Bash(git fetch *) Bash(git log *) Bash(git diff *) Bash(git grep *) Bash(git ls-files *) Bash(git merge-base *) Bash(git switch -c *) Bash(git switch --no-track -c *) Bash(git switch main) Bash(git switch master) Bash(git add -A) Bash(git add -- *) Bash(git reset -q -- *) Bash(git commit *) Bash(git cherry-pick *) Bash(git push --recurse-submodules=check -u origin HEAD) Bash(git pull --ff-only *) Bash(git branch -vv) Bash(git branch -d *) Bash(git branch -D *) Bash(git branch -f *) Bash(gh auth status *) Bash(gh pr view *) Bash(gh pr list *) Bash(gh pr create *) Bash(gh pr merge * --squash --delete-branch --match-head-commit *) Bash(gh pr merge * --merge --delete-branch --match-head-commit *) Bash(gh pr merge * --rebase --delete-branch --match-head-commit *) Bash(gh pr merge * --squash --match-head-commit *) Bash(gh pr checks *) Bash(git merge --no-edit origin/*) Bash(git merge --abort) Bash(npm ci) Bash(npm run * --if-present) Bash(python -m pytest -q) Bash(pbir validate *) Bash(pbir desktop list *)
---

# Ship

**Subagents and workflow agents: stop, the dispatching session commits.**

Take the work to a pushed feature branch with an open PR, in the repo that holds the changed
files. With changes in several repos, run the steps once per repo.

Requested: "$ARGUMENTS". The **mode** is `merge` only when the first word is exactly `merge`: then
land the PR too. Otherwise stop once the PR is open. Every other word is guidance for naming and
describing the change.

Steps 1 to 4 also serve an **auto-commit rule**: a CLAUDE.md instruction to follow them after
every finished task. Where a step differs under that rule, it says so.

Branch names, commit messages and the PR are read by whoever can read the repo: keep local paths,
hostnames, client names and anything the quarantine kept out of all of them.

Two files beside this one hold the rarer branches. Read one only when a step sends you there:
`references/edge-cases.md` (no or empty origin, the stop list in full, branches with a merged or
closed PR) and `references/landing.md` (step 6 detail, local checks, refusals).

## Ground rules

- Run every command through the **Bash tool** (Git Bash on Windows) as a bare `git ...` or
  `gh ...` from the repo root. With no Bash tool, stop and say so: a PowerShell pipe can prefix a
  BOM to the commit message.
- If the shell is elsewhere, `cd "<repo root>"` in a Bash call of its own first: the pre-approved
  commands match neither `git -C` nor `cd ... && git`. `Shell cwd was reset` means the repo is
  outside this session's directories: ask the user to `/add-dir` it and run the command again.
- Give `git commit`, `git cherry-pick`, `git push` and `git pull` the 600000 ms timeout. A timeout
  is a stop-and-report.
- When git refuses a command to protect local changes, stop and report. The forcing flags (`-f`,
  `--discard-changes`, `--merge`) are the user's call; a half-done cherry-pick gets
  `git cherry-pick --abort` first. Commit hooks always run.
- After any stop, the user's reply clears the pre-approved commands. Under `/ship`, ask them to
  rerun the same command (`/ship merge` after a stop in step 6), with their answer as guidance
  words. Under the auto-commit rule, resume at step 1.

## 1. Read the repo

```bash
git rev-parse --show-toplevel && git status && git remote -v && git log --oneline -8
```

With an `origin` remote:

- `git fetch --prune origin`, then `git remote set-head origin --auto`, then
  `git rev-parse --abbrev-ref origin/HEAD`. The **default branch** is that answer minus `origin/`.
  If the fetch fails (offline), skip `set-head` and carry on; the push will report it.
- `<url>` is `git remote get-url origin`; an SSH alias host (`git@github-work:owner/repo`) means
  github.com, so use `<owner>/<repo>`. The repo is **gh-ready** when
  `gh auth status --active --hostname <host>` succeeds.
- Pass `-R <url>` to every `gh pr` command: without it gh targets an `upstream` remote when one
  exists, which is someone else's repo.
- On a non-default branch, when gh-ready, **this branch's PR** comes from
  `gh pr view <branch> -R <url> --json number,url,state,headRefOid`. "No pull requests found"
  means it has none.

No `origin`, or an empty one (`No commits yet`, or `git log -1 --remotes=origin` prints nothing):
follow `references/edge-cases.md`.

Stop and report when a merge, rebase, cherry-pick, revert or am is in progress or there are
`Unmerged paths`; HEAD is detached; git warns `Filename too long`; `--show-toplevel` is a home
directory or a drive root; the branch name is all digits; or there is nothing to ship. The
`references/edge-cases.md` stop list says exactly what each means, and when `merge` mode skips to
step 6.5 instead.

Done when you know the default branch and, with an `origin`, `<url>` and whether the repo is
gh-ready, and `--show-toplevel` is the repo that holds the changed files.

## 2. Desktop guard

Applies when `git ls-files -co --exclude-standard -- ':/*.pbip' ':/*.pbir'` lists anything and
`pbir` is installed. Run `pbir desktop list --json`:

- `status: not_connected`: no Desktop is reachable. Carry on, and say so in the report.
- An instance holds this repo when its `currentFilePath` or `reportDir` sits under the repo root
  (case-insensitive, `\` read as `/`). With `hasUnsavedChanges: true`, stop. If this task edited
  that report on disk, tell the user a plain save overwrites those edits unless Desktop applies
  them first ("Apply external changes"), which drops the unsaved canvas edits; the user chooses,
  and you record which. Otherwise ask the user to save.
- A non-empty `connectionErrors`, or `status: partial`: ask once whether Desktop has this report
  open with unsaved changes.

Ship anyway only when the user says so.

## 3. Branch

New branches are `<type>/<slug>`: type `feat`, `fix`, `docs` or `chore`, a slug of 2 to 5
kebab-case words. The name must be new locally, on origin (no `origin/<name>`) and, when
gh-ready, as a PR head (`gh pr list -R <url> --head <name> --state all` returns nothing). No
branch named just `<type>` may exist locally or on origin.

- **On the default branch:** list `git log --oneline origin/<default>..<default>`; those unpushed
  commits ride in this PR and the report names them (if adding `--not --remotes` lists fewer,
  stop and ask: the rest are on another remote). Then `git switch -c <type>/<slug>`
  (uncommitted changes come along) and, when `origin/<default>` exists,
  `git branch -f <default> origin/<default>`.
- **On a feature branch whose PR is MERGED or CLOSED, a long-lived or PR-base branch, a branch
  with a gone or foreign upstream, or one cut from another shared branch:** follow the step 3
  sections of `references/edge-cases.md`. Most of them stop and ask.
- **On any other feature branch:** stay on it.

Done when HEAD is a branch other than the default, with no merged or closed PR.

## 4. Commit

Stage everything with `git add -A` under `/ship`. Under the auto-commit rule, stage only the files
this task changed (`git add -- <paths>`); a file that already had uncommitted edits before the
task stays unstaged. A repo with no `.gitignore` gets one first, covering its toolchain (build
output, dependency folders, for PBIP `**/.pbi/localSettings.json` and `**/.pbi/cache.abf`) plus
`.claude/worktrees/`; the report names it.

Read `git diff --cached --stat=400` and `git diff --cached --summary`. A conflict block that
`git diff --cached --check` reports (`<<<<<<<` through `>>>>>>>`) is a stop-and-report; a lone
`=======` heading underline is not.

Run the **secret scan**, `git diff --cached -i -G'<pattern>' --name-only`, with:

```
(api[_-]?key|access[_-]?key|account[_-]?key|subscription[_-]?key|sig=|secret|token|passw|pwd *[=:"]|bearer|authorization|private key|private-lines|"auth"|://[^/@ ]+:[^/@ ]+@|gh[pousr]_[a-z0-9]{30}|github_pat_|xox[abposr]-|hooks\.slack\.com|sk_live_|sk-(proj|ant)-|eyJ[a-z0-9_-]{10,}\.eyJ|[?&]code=)
```

Read the matching lines of every listed file with
`git grep --cached -n -i -E '<pattern>' -- <paths>`, opening the full diff only where a line
leaves doubt.

Unstage (`git reset -q -- <path>`) anything in the **quarantine**, then confirm with
`git diff --cached --name-only` that each is gone (a non-ASCII path must be passed decoded):

- secrets: `.env*` (not `.env.example`), `*.pem`, `*.key`, `*.pfx`, files named for credentials
  or tokens, and any file whose lines hold a real key, token, password or connection string
- PBIP local state: `**/.pbi/localSettings.json`, `**/.pbi/cache.abf`, and a `*.pbix` new to the
  repo
- embedded repos: `create mode 160000 <path>` for a path not in `.gitmodules`
- new files of a data or opaque type the repo does not already track: `*.csv`, `*.tsv`, `*.xlsx`,
  `*.parquet`, `*.db`, `*.sqlite`, `*.bak`, `*.dump`, `*.har`, `*.log`, or a `Bin 0 -> <n> bytes`
  file that is not an image, font or PDF
- files over 10 MB: read `git ls-files --format='%(objectsize) %(path)' -- <staged paths>`

Report each quarantined path with the `.gitignore` line that would keep it out.

One commit per logical change; split unrelated staged work by path. Copy the style of the repo's
recent `git log`: a subject under 72 characters, a body only when the reason is not obvious.
Commit through a quoted heredoc so `$` and backticks survive:

```bash
git commit -F - <<'SHIP_MSG_EOF'
Subject line

Optional body.
SHIP_MSG_EOF
```

A failing hook is a stop-and-report, unless it only rewrote files: re-stage them and commit once
more.

Done when `git status --short --untracked-files=all` shows nothing staged, and nothing unstaged
except quarantined paths, submodule changes and (under the auto-commit rule) edits that predate
this task, each listed in the report.

## 5. Push and open the PR

No `origin`: stop and report a local-only repo. If `git log origin/<default>..HEAD` is empty, the
quarantine kept everything out: stop and report (an empty origin skips this check).

Scan every commit the push carries, `HEAD --not --remotes=origin`: run
`git log -i -G'<pattern>' --format=%h --name-only HEAD --not --remotes=origin` and read each
listed commit's matching lines with `git grep -n -i -E '<pattern>' <sha> -- <its paths>` (a
secret one commit adds and a later one removes is still pushed); and read
`git log --stat=400 --summary --format= HEAD --not --remotes=origin` with sizes from
`git ls-files --format='%(objectsize) %(path)'`. A real secret, or a path the quarantine matches,
is a stop-and-report. Then `git push --recurse-submodules=check -u origin HEAD`; a rejected push
is a stop-and-report.

Empty origin (a first push), or not gh-ready: stop after the push and report.

When this branch's PR is OPEN it now carries the new commits. Otherwise create one, then read
`<number>` and `<pr url>` with `gh pr view <branch> -R <url> --json number,url,state`.
Single-quote the title (an apostrophe is `'\''`). The body follows this template and no other:

```bash
gh pr create -R <url> --base <default> --head <branch> --title '<subject>' --body-file - <<'SHIP_PR_EOF'
## Summary
- what changed and why, one bullet per logical change

## Verified
- what was actually checked (build, tests, a screenshot), or "Not verified"

## Merge Danger
**Door:** one-way or two-way
**Blast radius:** one word, then a line on what could break if it is wrong
SHIP_PR_EOF
```

**Door** is two-way when a revert fully undoes the PR, one-way when it does something a revert
cannot take back (deletes or migrates data, renames a shared surface, deploys, emails, rewrites
history). **Blast radius** is who feels a mistake: `none`, `local`, `report`, `model`,
`consumers`, `client`.

Done when `gh pr view <branch> -R <url> --json number,url,state` returns an OPEN PR, or an early
stop above is reported and any push it made succeeded.

## 6. Land (`merge` mode only)

Nothing lands until the PR's head contains the current default branch, GitHub sees no conflict,
and every check on that exact head has passed. 6.1 to 6.3 are that gate; a stop anywhere in it
leaves the PR open and unmerged. Read `references/landing.md` when any command here does
anything but succeed.

1. **Up to date.** `git fetch origin <default>`, then
   `git merge-base --is-ancestor origin/<default> HEAD`. If that fails, merge the default branch
   in as `references/landing.md` 6.1 says; a conflict is a stop-and-report.
2. **No conflict on GitHub.** `gh pr view <pr url> -R <url> --json headRefOid,isDraft,mergeable`.
   `headRefOid` must equal `git rev-parse HEAD` (otherwise someone else pushed: stop). A draft or
   `CONFLICTING` is a stop-and-report; `UNKNOWN` is read again at the end of 6.3.
3. **Every check passed.** `gh pr checks <pr url> -R <url> --watch --fail-fast --interval 30`,
   then `gh pr checks <pr url> -R <url> --json name,bucket,link`: every `bucket` must be `pass`
   or `skipping`, and anything else is a stop-and-report naming the check and its link.
   `no checks reported` means no CI: run the local checks in `references/landing.md` instead.
   Then `mergeable` must say `MERGEABLE`, `headRefOid` must not have moved, and the default
   branch must not have moved (`git fetch origin <default>`; if it did, back to 6.1).
4. **Merge**, pinned to the head that passed:
   `gh pr merge <pr url> -R <url> --squash --delete-branch --match-head-commit <headRefOid>`.
   Read `gh pr view <pr url> -R <url> --json state`: MERGED, or see `references/landing.md`.
5. Run the Desktop guard again, then `git switch <default> && git pull --ff-only origin <default>`.
6. Prune: `git fetch --prune`, then delete each branch `git branch -vv` marks gone, as
   `references/landing.md` says.

Done when the gate passed on the merged head, the PR says MERGED, HEAD is the updated default
branch, and every gone branch whose tip is a merged PR's head is deleted (or listed).

## 7. Report

Branch, commits (short sha and subject), unpushed default-branch commits that rode along, PR URL
and state, quarantined paths, edits left unstaged, and every step skipped with its reason. In
`merge` mode, also the gate: any merge of the default branch 6.1 made, and each check with its
result (or each local command run instead). Read the PR state from `gh pr view`, not from memory.
If files under a report that Desktop holds changed on disk, tell the user to accept Desktop's
"Apply external changes" banner.
