# Ship: step 6 in detail

Read by [SKILL.md](../SKILL.md) when a step 6 command does anything other than succeed, and for
the local checks a repo with no CI needs.

## 6.1 The default branch moved on

When `git merge-base --is-ancestor origin/<default> HEAD` fails, the checks so far ran against an
older base:

- Run the Desktop guard again, since the merge rewrites files. Then
  `git merge --no-edit origin/<default>`.
- A conflict is a stop-and-report: list the paths from `git diff --name-only --diff-filter=U`,
  then `git merge --abort`. Never resolve a conflict under this skill, and never rebase or
  force-push a pushed branch to get round one.
- After a clean merge, push (`git push --recurse-submodules=check -u origin HEAD`) so the checks
  run again on the merged result.

## 6.3 Checks

- `gh pr checks <pr url> -R <url> --watch --fail-fast --interval 30` with the 600000 ms timeout.
  On a timeout run it once more, then stop and report.
- Checks that branch protection does not require count too: a red optional check still stops
  the merge.
- If the default branch moved while the checks ran (`git fetch origin <default>` once more), go
  back to 6.1. A third time round is a stop-and-report: the branch cannot keep up with a busy
  default branch, and the user decides.

**Local checks, only when the repo has no CI** (`no checks reported`). From the repo root, run
what the repo itself defines, and require exit 0 from every command:

- `package.json`: `npm ci` when `node_modules` is missing, then `npm run <name> --if-present`
  for `typecheck`, `lint`, `test` and `build`, in that order.
- A Python project with a pytest configuration: `python -m pytest -q`.
- A PBIP repo with pbir installed: `pbir validate "<path>.Report"` for each report the PR changes.
- A repo that defines no checks at all: stop and ask. Merging unchecked work is the user's call.

## 6.4 When the merge command refuses

- GitHub refuses squash merges: use `--merge`, or `--rebase` if that is refused too, and say
  which.
- gh answers `Cannot use -d or --delete-branch when merge queue enabled`: run it again without
  `--delete-branch`.
- Any other refusal (conflicts, required checks, missing reviews, a draft, a moved head) is a
  stop-and-report; quote its message.
- gh prints nothing on success here, so read `gh pr view <pr url> -R <url> --json state`. OPEN
  means queued: report that and stop, and a later `/ship merge` finishes 6.5 and 6.6.
- Merges run without `--admin` or `--auto`. `<pr url>` carries GitHub's own spelling of the
  owner, which `--delete-branch` needs. With `-R`, gh merges and deletes the remote branch and
  leaves this checkout alone.

## 6.5 and 6.6 Switching back and pruning

- If git says the default branch is already used by a worktree, skip the switch and say so.
- Prune every local branch that `git branch -vv` marks `[origin/<branch>: gone]` with
  `git branch -d`. When `-d` refuses (a squash merge looks unmerged to git), use `git branch -D`
  only when
  `gh pr list -R <url> --head <branch> --state merged --json headRefOid --jq '.[].headRefOid'`
  prints the sha from `git rev-parse <branch>`. Otherwise keep the branch and list it. A branch
  that a worktree has checked out stays, and is listed.
