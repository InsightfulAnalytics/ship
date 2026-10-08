---
name: ship
description: "Ship the current work: branch off the default branch if needed, commit, push, open a PR. `/ship merge` also lands it once it is up to date, conflict-free and every check has passed, then prunes."
argument-hint: "[merge] [notes]"
disable-model-invocation: true
allowed-tools: Bash(git status *) Bash(git rev-parse *) Bash(git remote -v) Bash(git remote get-url *) Bash(git remote set-head *) Bash(git fetch *) Bash(git log *) Bash(git diff *) Bash(git grep *) Bash(git ls-files *) Bash(git merge-base *) Bash(git switch -c *) Bash(git switch --no-track -c *) Bash(git switch main) Bash(git switch master) Bash(git add -A) Bash(git add -- *) Bash(git reset -q -- *) Bash(git commit *) Bash(git cherry-pick *) Bash(git push --recurse-submodules=check -u origin HEAD) Bash(git pull --ff-only *) Bash(git branch -vv) Bash(git branch -d *) Bash(git branch -D *) Bash(git branch -f *) Bash(gh auth status *) Bash(gh pr view *) Bash(gh pr list *) Bash(gh pr create *) Bash(gh pr merge * --squash --delete-branch --match-head-commit *) Bash(gh pr merge * --merge --delete-branch --match-head-commit *) Bash(gh pr merge * --rebase --delete-branch --match-head-commit *) Bash(gh pr merge * --squash --match-head-commit *) Bash(gh pr checks *) Bash(git merge --no-edit origin/*) Bash(git merge --abort) Bash(npm ci) Bash(npm run * --if-present) Bash(python -m pytest -q) Bash(pbir validate *) Bash(pbir desktop list *)
---

# Ship

Take the work to a pushed feature branch with an open PR, in the repo that holds the changed
files. With changes in several repos, run the steps once per repo.

Requested: "$ARGUMENTS". The **mode** is `merge` only when the first word is exactly `merge`: then
land the PR too. Otherwise stop once the PR is open. Every other word is guidance for naming and
describing the change.

The same steps 1 to 4 also serve an **auto-commit rule**: a CLAUDE.md instruction to follow them
after every finished task. Where a step differs under that rule, it says so.

Branch names, commit messages and the PR are read by whoever can read the repo: keep local paths,
hostnames, client names and anything the quarantine kept out of all of them.

## Ground rules

- Run every command through the **Bash tool** (Git Bash on Windows) as a bare `git ...` or
  `gh ...` from the repo root. With no Bash tool (Git Bash not found), stop and say so: a
  PowerShell pipe can prefix a BOM to the commit message.
- If the shell is elsewhere, `cd "<repo root>"` in a Bash call of its own first: the pre-approved
  commands match neither `git -C` nor `cd ... && git`. If the result says `Shell cwd was reset`,
  the repo is outside this session's directories: ask the user to `/add-dir` it and run the
  command again.
- Give `git commit`, `git cherry-pick`, `git push` and `git pull` the Bash tool's 600000 ms
  timeout. A timeout is a stop-and-report.
- When git refuses a command to protect local changes (a switch that would overwrite files, a
  cherry-pick conflict, a pull that cannot fast-forward), stop and report. The forcing flags
  (`-f`, `--discard-changes`, `--merge`) are the user's call; a half-done cherry-pick gets
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
  If the fetch fails (offline), skip `set-head` and carry on with the refs already there; the push
  in step 5 will report it.
- `<url>` is `git remote get-url origin` and `<host>` is its host. An SSH alias host
  (`git@github-work:owner/repo`) means github.com: use `<owner>/<repo>` as `<url>` below. The repo
  is **gh-ready** when `gh auth status --active --hostname <host>` succeeds.
- Pass `-R <url>` to every `gh pr` command, with a branch, PR number or PR URL as the argument.
  Without `-R`, gh targets an `upstream` remote when one exists, which is someone else's repo.
- On a non-default branch, when gh-ready, **this branch's PR** comes from
  `gh pr view <branch> -R <url> --json number,url,state,headRefOid`: `<number>`, `<pr url>`, its
  state and `<headRefOid>`. "No pull requests found" means it has none.

**No `origin` remote:** the default branch is whichever of `main` or `master` exists. Skip step 3
and commit on the current branch, since nothing could ever land a feature branch.

**Empty origin or first commit:** `git status` says `No commits yet`, or
`git log -1 --remotes=origin` prints nothing after the fetch. The default branch is local `main`
or `master` if one exists, else the current branch. On it, skip step 3 and commit there. On any
other branch, stop and ask: the first branch pushed to an empty GitHub repo becomes its default.
If `set-head --auto` fails while origin does have branches, stop and ask which is the default.

Stop and report, rather than working around it, when:

- `git status` names a merge, rebase, cherry-pick, revert or am in progress, or lists
  `Unmerged paths`
- HEAD is detached
- git warns `Filename too long`: files under that path are invisible until the user runs
  `git config core.longpaths true`
- `--show-toplevel` is a home directory or a drive root
- the branch name is all digits: gh reads `gh pr view 123` as PR #123, so ask for a rename
- there is nothing to ship: `git status --short` lists only quarantine paths (step 4) and
  changes inside submodules (` m` or ` ?` on a submodule path), and either
  `git log origin/<default>..HEAD` is empty or this branch's PR is MERGED and
  `git log --no-merges <headRefOid>..HEAD ^origin/<default>` is empty. With no `origin`, such a
  tree alone. With an empty origin, only under `No commits yet`, since any local commit is
  unpushed. In `merge` mode, when this branch's PR is MERGED and nothing is new, skip to step 6.5
  instead, whichever clause matched.

Wherever `<headRefOid>` is missing from this clone (git says `Invalid revision range` or
`Not a valid commit name`), run `git fetch origin pull/<number>/head` first: GitHub keeps that ref
after the branch is deleted.

Done when you know the default branch and, with an `origin`, `<url>` and whether the repo is
gh-ready, and `--show-toplevel` is the repo that holds the changed files.

## 2. Desktop guard

Applies when `git ls-files -co --exclude-standard -- ':/*.pbip' ':/*.pbir'` lists anything and
`pbir` is installed. Run `pbir desktop list --json`:

- `status: not_connected` (exit 1): no Desktop is reachable, because it is closed or its local-API
  preview is off. Carry on, and say so in the report.
- An instance holds this repo when its `currentFilePath` or `reportDir` sits under the repo root
  (compare case-insensitively, reading `\` as `/`). With `hasUnsavedChanges: true` it has canvas
  changes that are not on disk, so a commit would ship a stale report.
- A non-empty `connectionErrors`, or `status: partial`: some instance could not be read. Ask once
  whether Desktop has this report open with unsaved changes.

When an instance holds this repo with unsaved changes, stop:

- If this task edited files under that report on disk, tell the user that a plain save overwrites
  them unless Desktop has applied them first (its "Apply external changes" banner), and that
  applying drops the unsaved canvas edits. The user chooses; record which.
- Otherwise ask the user to save.

Ship anyway only when the user says so.

Done when no instance under this repo shows `hasUnsavedChanges: true`, and any unreadable instance
was cleared with the user.

## 3. Branch

New branches are named `<type>/<slug>`: type `feat`, `fix`, `docs` or `chore`, and a slug of 2 to 5
kebab-case words naming the change. The name must be new as a local branch, as a remote branch
(no `origin/<name>` after the fetch) and, when gh-ready, as a PR head
(`gh pr list -R <url> --head <name> --state all` returns nothing). No branch named just `<type>`
may exist locally or on origin, because git cannot hold both `docs` and `docs/x`.

- **On the default branch:** list `git log --oneline origin/<default>..<default>`. Those unpushed
  commits ride in this PR, and the report names them. If adding `--not --remotes` lists fewer,
  the rest are already on another remote (the branch was synced from `upstream`): stop and ask.
  Then `git switch -c <type>/<slug>` (uncommitted changes come along) and, when
  `origin/<default>` exists, `git branch -f <default> origin/<default>`, so the default branch
  matches the remote and cannot diverge from the squash merge later.
- **On a feature branch whose PR is MERGED** (step 1): commit any uncommitted work on it first
  (step 4). If `git merge-base --is-ancestor <headRefOid> <branch>` fails, GitHub moved the head.
  When `git log --no-merges --oneline <branch>..<headRefOid> ^origin/<default>` is empty, GitHub
  only merged the default branch in (its "Update branch" button): carry on. Otherwise the head was
  rewritten: stop and list `git log --oneline origin/<default>..<branch>` for the user. To carry
  on, `git switch --no-track -c <type>/<slug> origin/<default>` and
  `git cherry-pick --no-merges <headRefOid>..<branch> ^origin/<default>`. If the cherry-pick stops,
  abort it, `git switch <branch>`, `git branch -D <type>/<slug>`, and stop.
- **Stop and ask** on any of these:
  - a feature branch whose PR is CLOSED, or whose upstream is gone with no merged PR found: its
    commits are not on `origin/<default>`
  - a long-lived branch (`develop`, `dev`, `staging`, `release/*`, `gh-pages`) or the base of an
    open PR (`gh pr list -R <url> --base <branch> --state open` lists one): landing it would
    squash it into the default and delete it
  - a branch cut from another shared branch:
    `git log --oneline HEAD --not origin/<default> --exclude=origin/<branch> --remotes=origin`
    lists fewer commits than `git log --oneline origin/<default>..HEAD`
  - a branch that tracks another remote's branch of the same name, or whose upstream git cannot
    resolve (`git rev-parse --abbrev-ref @{upstream}` says
    `not stored as a remote-tracking branch`): that is how `gh pr checkout` leaves someone else's
    PR, and a push would copy it into origin as a second PR
- **On any other feature branch:** stay on it.

Done when HEAD is a branch other than the default, with no merged or closed PR.

## 4. Commit

Stage everything with `git add -A` under `/ship`. Under the auto-commit rule, stage only the files
this task changed (`git add -- <paths>`); a file that already had uncommitted edits before the
task stays unstaged. A repo with no `.gitignore` gets one first, covering its toolchain (build
output, dependency folders, and for PBIP `**/.pbi/localSettings.json` and `**/.pbi/cache.abf`,
keeping the `**/` because each `.SemanticModel` and `.Report` folder has its own `.pbi/`) plus
`.claude/worktrees/`. It goes in the commit and the report names it.

Read `git diff --cached --stat=400` and `git diff --cached --summary`. If
`git diff --cached --check` reports a `leftover conflict marker`, read those lines: a conflict
block (`<<<<<<<` through `>>>>>>>`) is a stop-and-report, while a lone `=======` heading underline
is not one.

Run the **secret scan**, `git diff --cached -i -G'<pattern>' --name-only`, with this pattern:

```
(api[_-]?key|access[_-]?key|account[_-]?key|subscription[_-]?key|sig=|secret|token|passw|pwd *[=:"]|bearer|authorization|private key|private-lines|"auth"|://[^/@ ]+:[^/@ ]+@|gh[pousr]_[a-z0-9]{30}|github_pat_|xox[abposr]-|hooks\.slack\.com|sk_live_|sk-(proj|ant)-|eyJ[a-z0-9_-]{10,}\.eyJ|[?&]code=)
```

Read the matching lines of every file it lists,
`git grep --cached -n -i -E '<pattern>' -- <paths>`, and open the full diff only where a line
leaves doubt.

Unstage (`git reset -q -- <path>`) anything in the **quarantine**, then confirm with
`git diff --cached --name-only` that each is gone: `git reset` exits 0 on a path that matches
nothing, and a non-ASCII path must be passed decoded, not in the octal-escaped form git prints.

- secrets: `.env*` (not `.env.example`), `*.pem`, `*.key`, `*.pfx`, files named for credentials
  or tokens, and any file whose lines hold a real key, token, password or connection string
- PBIP local state: `**/.pbi/localSettings.json`, `**/.pbi/cache.abf`, and a `*.pbix` new to the
  repo (`A` in `git diff --cached --name-status`)
- embedded repos: a `create mode 160000 <path>` line in the summary for a path not in
  `.gitmodules`, usually `.claude/worktrees/*`
- new files of a data or opaque type the repo does not already track: `*.csv`, `*.tsv`, `*.xlsx`,
  `*.parquet`, `*.db`, `*.sqlite`, `*.bak`, `*.dump`, `*.har`, `*.log`, or a `Bin 0 -> <n> bytes`
  file that is not an image, font or PDF (the scan cannot read binaries, and UTF-16 text counts
  as binary)
- files over 10 MB, binary or text: read
  `git ls-files --format='%(objectsize) %(path)' -- <staged paths>`, since the stat shows a
  19 MB one-line JSON as `1 +`

Report each quarantined path with the `.gitignore` line that would keep it out.

Make one commit per logical change: when the staged work spans unrelated changes, split it by
path. Copy the message style of the repo's recent `git log`: a subject under 72 characters, and a
body only when the reason is not obvious from the diff. Commit through a quoted heredoc so `$` and
backticks survive:

```bash
git commit -F - <<'SHIP_MSG_EOF'
Subject line

Optional body.
SHIP_MSG_EOF
```

A failing commit hook is a stop-and-report, unless it only rewrote files (a formatter): then
re-stage those files and commit once more.

Done when `git status --short --untracked-files=all` shows nothing staged, and nothing unstaged
except quarantined paths, changes inside submodules, and (under the auto-commit rule) edits that
predate this task, each listed in the report.

## 5. Push and open the PR

No `origin`: stop here and report a local-only repo.

If `git log origin/<default>..HEAD` is empty, the quarantine kept everything out: stop and report.
An empty origin skips this check.

Scan every commit the push will carry, the range `HEAD --not --remotes=origin`:

- `git log -i -G'<pattern>' --format=%h --name-only HEAD --not --remotes=origin`, then read each
  listed commit's matching lines as that commit wrote them,
  `git grep -n -i -E '<pattern>' <sha> -- <its paths>`: a secret one commit adds and a later one
  removes is still pushed.
- `git log --stat=400 --summary --format= HEAD --not --remotes=origin`, plus
  `git ls-files --format='%(objectsize) %(path)' -- <paths it lists>` for sizes (the index
  equals HEAD after step 4): a path the quarantine matches by name, type or size is a
  stop-and-report, since unstaging cannot take it out of a commit (and a commit hook can re-add
  one).

A real secret is a stop-and-report. With anything in that range,
`git push --recurse-submodules=check -u origin HEAD`. A rejected push is also a stop-and-report:
the fix is the user's call.

Empty origin (a first push), or not gh-ready: stop after the push and report.

When this branch's PR is OPEN, it now carries the new commits. Otherwise create one, then read its
`<number>` and `<pr url>` with `gh pr view <branch> -R <url> --json number,url,state`.
Single-quote the title, writing an apostrophe as `'\''`. The body follows this template and no
other; ignore any other skill's PR-body template:

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

**Door:** two-way when reverting the PR fully undoes it; one-way when it does something a revert
cannot take back (deletes or migrates data, renames a published or shared surface, deploys, emails,
rewrites history). **Blast radius:** who or what feels a mistake, e.g. `none`, `local`, `report`,
`model`, `consumers`, `client`. Keep both to a line each.

Done when `gh pr view <branch> -R <url> --json number,url,state` returns an OPEN PR, or, on the
early stops above, when the stop is reported and any push it made succeeded.

## 6. Land (`merge` mode only)

With an empty mode, go to the report.

Nothing lands on the default branch until the PR's head already contains the current default
branch, GitHub sees no conflict, and every check on that exact head has passed. Steps 6.1 to 6.3
are that gate. A stop anywhere in it leaves the PR open and unmerged.

1. **Up to date.** `git fetch origin <default>`, then
   `git merge-base --is-ancestor origin/<default> HEAD`. When that fails, the default branch has
   moved on since this branch left it, and the checks so far ran against an older base:
   - Run the Desktop guard again, since the merge rewrites files. Then
     `git merge --no-edit origin/<default>`.
   - A conflict is a stop-and-report: list the paths from `git diff --name-only --diff-filter=U`,
     then `git merge --abort`. Never resolve a conflict under this skill, and never rebase or
     force-push a pushed branch to get round one.
   - After a clean merge, push (`git push --recurse-submodules=check -u origin HEAD`) so the
     checks run again on the merged result.
2. **No conflict on GitHub.** Read
   `gh pr view <pr url> -R <url> --json headRefOid,isDraft,mergeable`.
   - `headRefOid` must equal `git rev-parse HEAD`. Anything else means somebody else pushed:
     stop and report.
   - A draft, or `mergeable: CONFLICTING`, is a stop-and-report.
   - `mergeable: UNKNOWN` means GitHub is still working it out: read it again at the end of 6.3.
3. **Every check passed.** Wait with
   `gh pr checks <pr url> -R <url> --watch --fail-fast --interval 30` and the 600000 ms timeout.
   On a timeout run it once more, then stop and report. Then read
   `gh pr checks <pr url> -R <url> --json name,bucket,link`:
   - Every check's `bucket` must be `pass` or `skipping`. A `fail`, `cancel` or `pending` check is
     a stop-and-report that names the check and its link. Checks that branch protection does not
     require count too: a red optional check still stops the merge.
   - `no checks reported` means the repo has no CI for this branch. Run its own checks on HEAD
     instead (below).
   - Read 6.2's fields again: `mergeable` must now say `MERGEABLE`, and `headRefOid` must not have
     moved.
   - `git fetch origin <default>` once more. If the default branch moved while the checks ran,
     go back to 6.1. A third time round is a stop-and-report: the branch cannot keep up with a
     busy default branch, and the user decides.

   **Local checks, only when the repo has no CI.** From the repo root, run what the repo itself
   defines, and require exit 0 from every command:
   - `package.json`: `npm ci` when `node_modules` is missing, then `npm run <name> --if-present`
     for `typecheck`, `lint`, `test` and `build`, in that order.
   - A Python project with a pytest configuration: `python -m pytest -q`.
   - A PBIP repo with pbir installed: `pbir validate "<path>.Report"` for each report the PR
     changes.
   - A repo that defines no checks at all: stop and ask. Merging unchecked work is the user's
     call.
4. **Merge** by `<pr url>`, which carries GitHub's own spelling of the owner that
   `--delete-branch` needs, pinned to the head that passed the gate:
   `gh pr merge <pr url> -R <url> --squash --delete-branch --match-head-commit <headRefOid>`.
   With `-R`, gh merges and deletes the remote branch and leaves this checkout alone.
   - GitHub refuses squash merges: use `--merge`, or `--rebase` if that is refused too, and say
     which.
   - gh answers `Cannot use -d or --delete-branch when merge queue enabled`: run it again without
     `--delete-branch`.
   - Any other refusal (conflicts, required checks, missing reviews, a draft, a moved head) is a
     stop-and-report; quote its message.
   - gh prints nothing on success here, so read `gh pr view <pr url> -R <url> --json state`. OPEN
     means queued: report that and stop, and a later `/ship merge` finishes 6.5 and 6.6.
   - Merges run without `--admin` or `--auto`.
5. Run the Desktop guard again, since the next commands rewrite files under Desktop. Then
   `git switch <default> && git pull --ff-only origin <default>`. If git says the default branch
   is already used by a worktree, skip this and say so.
6. Prune: `git fetch --prune`, then take every local branch that `git branch -vv` marks gone
   (`[origin/<branch>: gone]`). Delete it with `git branch -d`. When `-d` refuses (a squash merge
   looks unmerged to git), use `git branch -D` only when
   `gh pr list -R <url> --head <branch> --state merged --json headRefOid --jq '.[].headRefOid'`
   prints the sha from `git rev-parse <branch>`. Otherwise keep the branch and list it.

Done when the gate passed on the head that was merged, `gh pr view <pr url> -R <url> --json state`
says MERGED, HEAD is the updated default branch (or the worktree skip was reported), and every
gone branch whose tip is a merged PR's head is deleted, apart from one git refuses because a
worktree has it checked out (listed).

## 7. Report

Branch, commits (short sha and subject), unpushed default-branch commits that rode along, PR URL
and state, quarantined paths, edits left unstaged, and every step skipped with its reason. In
`merge` mode, also the gate: any merge of the default branch 6.1 made, and each check with its
result (or each local command run instead). Read the PR state from `gh pr view`, not from memory.
If files under a report that Desktop holds changed on disk (task edits, step 3, step 6.1,
step 6.5), tell the user to accept Desktop's "Apply external changes" banner.
