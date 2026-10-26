---
name: land
description: >-
  Land the pending change in the utils repository (github.com/codegod100/utils):
  commit any uncommitted work on main, verify it with the repo's pixi tasks, push
  main to origin, and confirm origin/main carries the commit. Invoke only when
  the user has explicitly requested landing the change — the Land Changes
  button, an explicit `/land`, or wording such as "push this" or "land it".
  Never invoke it for review, preparation, running checks, or skill installation.
disable-model-invocation: true
metadata:
  delta-action: land
---

# Land changes in `utils`

This repository lands by pushing `main` directly: no pull requests, no CI
workflows, no branch protection. Landing is therefore commit, verify locally,
push, confirm.

The request that invoked this skill is the permission to land — do not ask again
whether to merge, and do not stop at "ready to push". Carry the workflow through
to a pushed and verified `origin/main`, or report a genuine blocker.

Run every command from this checkout's repository root.

## 1. Preflight

```sh
git remote -v                        # origin = https://github.com/codegod100/utils.git
git branch --show-current            # must be main
git status --porcelain               # uncommitted work to commit
```

- `origin` must be `github.com/codegod100/utils.git`. If it is not, stop: this
  skill only knows how to land this repository.
- The `local` remote points at the primary checkout (`~/Projects/utils`). Work
  only in the current checkout, and never pull, reset, checkout or commit in the
  directory `local` points to.
- If the branch is not `main`, stop and report; landing to another branch is not
  defined for this repository.
- If the push later needs credentials, they come from the existing git
  credential helper for github.com; never print or ask for a token.

## 2. Commit the pending work

Skip this step when `git status --porcelain` is empty.

```sh
git add -A .
GIT_EDITOR=true git commit -F - <<'MSG'
<subject>

<body>
MSG
```

Match the history's style (`git log --oneline`): `<tool>: <imperative summary>`
when the change belongs to one tool, a bare imperative subject when it spans the
repository. Use the body for the decision that is not visible in the diff, not
for a restatement of it. Never fabricate changelog, tests or issue references the
repository does not use.

## 3. Verify before pushing

Both must pass on the exact tree being pushed:

```sh
pixi run lint                    # pixi.toml [tasks] lint = "ruff check who-killed"
pixi run who-killed --help       # pixi.toml [tasks] who-killed
```

- `ruff` is only inside the pixi environment, so the check has to go through
  pixi; a bare `ruff` on `PATH` is a different, possibly absent, tool.
- The first `pixi run` in a fresh checkout installs `.pixi/envs/default` from
  `pixi.lock`. That download is expected to be slow, and is not a failure.
- `pixi run who-killed --help` is a smoke check that the environment and the
  extension-less tool still run; it needs no root.
- If `lint` fails on something the change introduced and the fix is obvious and
  in scope, fix it, re-run, and mention it in the final report. Otherwise stop:
  report that nothing landed and quote the failing output.
- This repository has no remote checks to wait for. If
  `gh api repos/codegod100/utils/actions/workflows --jq .total_count` ever
  returns more than `0`, wait for those runs on the pushed commit before
  reporting success.

## 4. Push

```sh
GIT_EDITOR=true git fetch origin
git rev-list --count origin/main..HEAD          # commits to land
git merge-base --is-ancestor origin/main HEAD || GIT_EDITOR=true git rebase origin/main
git push origin main
```

- Rebase onto a moved `origin/main` rather than merging, so the branch stays
  linear as the existing history is (`git log --oneline`).
- Conflicts: resolve them automatically when the intended result is clear — read
  both sides, keep the incoming behaviour and re-apply the change on top. Then
  `git add <paths>` and `GIT_EDITOR=true git rebase --continue`. Stop and report
  if the intent is genuinely ambiguous or the conflict is not mechanical.
- Re-run `pixi run lint` after any rebase: the verified tree must be the pushed
  tree.
- If the push is rejected as non-fast-forward, fetch, rebase, re-verify, and
  retry once.
- Never `--force` or `--force-with-lease`, never push another branch, never push
  tags, and never create a PR, release or branch: none of that is part of
  landing here.
- Never run `pixi run install` while landing. It repoints
  `~/.local/bin/who-killed` at whichever checkout runs it, and a landing
  checkout is not the place a `PATH` entry should point.

## 5. Confirm and report

```sh
git fetch origin
test "$(git rev-parse HEAD)" = "$(git rev-parse origin/main)"
git status -sb
git log --oneline -3
```

Report the outcome in the first line: landed, or not landed.

- On success name the commits that landed (for example `b9d2097..448a022`) and
  link `https://github.com/codegod100/utils/commit/<sha>` for the head commit.
  Say that local `main` and `origin/main` are level, and that the primary
  checkout still catches up when the user pulls.
- If there was nothing to land (clean tree, no commits ahead), say so explicitly
  and push nothing; never create an empty commit.
- If any gate blocked the landing, say the changes did **not** land, name the
  blocker, and quote the relevant command output. Do not describe a prepared or
  pushed-nowhere branch as success.

Landing this repository never needs root. If a step appears to require it, stop
and report instead of escalating.
