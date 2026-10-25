# Ship: edge cases for steps 1 and 3

Read by [SKILL.md](../SKILL.md) when step 1 finds no `origin`, an empty origin or a stop
condition, or when step 3 meets a feature branch that is not a plain one.

## Step 1: no origin, or an empty origin

**No `origin` remote:** the default branch is whichever of `main` or `master` exists. Skip step 3
and commit on the current branch, since nothing could ever land a feature branch.

**Empty origin or first commit:** `git status` says `No commits yet`, or
`git log -1 --remotes=origin` prints nothing after the fetch. The default branch is local `main`
or `master` if one exists, else the current branch. On it, skip step 3 and commit there. On any
other branch, stop and ask: the first branch pushed to an empty GitHub repo becomes its default.
If `set-head --auto` fails while origin does have branches, stop and ask which is the default.

## Step 1: the stop conditions in full

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

## Step 3: a feature branch whose PR is MERGED

Commit any uncommitted work on it first (step 4). If
`git merge-base --is-ancestor <headRefOid> <branch>` fails, GitHub moved the head. When
`git log --no-merges --oneline <branch>..<headRefOid> ^origin/<default>` is empty, GitHub only
merged the default branch in (its "Update branch" button): carry on. Otherwise the head was
rewritten: stop and list `git log --oneline origin/<default>..<branch>` for the user.

To carry on with new work after a merged PR,
`git switch --no-track -c <type>/<slug> origin/<default>` and
`git cherry-pick --no-merges <headRefOid>..<branch> ^origin/<default>`. If the cherry-pick stops,
abort it, `git switch <branch>`, `git branch -D <type>/<slug>`, and stop.

## Step 3: stop and ask

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
- on the default branch, when `git log --oneline origin/<default>..<default> --not --remotes`
  lists fewer commits than the same command without `--not --remotes`: the rest are already on
  another remote (the branch was synced from `upstream`)
