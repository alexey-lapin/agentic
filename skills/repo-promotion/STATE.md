# Plan state

Everything this skill produces lives under `.promotion/` in the repo root: `plan.md`, and
`drafts/<channel>.md` once step 11 writes any. One directory, so it is excluded once and
removed in one move.

## Where it lives

The repo the plan describes is the right place to keep it, and the plan's history is worth
as much as the plan: which item was dropped, when the ceiling moved, what a channel
actually returned. Both hold at once through an orphan-branch worktree.

Set it up on a first run, in a repo with a working tree:

```
git worktree add --orphan -b promotion .promotion   # git 2.42+
printf '.promotion/\n' >> "$(git rev-parse --git-path info/exclude)"
```

`promotion` shares no history with the default branch and can never merge into it.
`git merge-base <default-branch> promotion` finding no common ancestor is the check. The
work lives in the repo's own `.git`, so there is no second repository and no submodule,
and `git -C .promotion log` reads the plan's history like any other.

Exclude through `.git/info/exclude` rather than `.gitignore`. `.gitignore` is tracked, and
planning is not approved work.

Fall back to a plain excluded directory, no worktree, when git is older than 2.42, when
there is no working tree to hang one off, or when the user says so. Everything else here
still applies except the commits.

## Never push it

The plan carries ceiling estimates, candid reads on the repo's own weaknesses, and
assessments of other people's projects. Most repos are public. Publishing is a decision
the user makes, never a default, so leave `promotion` unpushed and say it exists.

Where the user wants it off the machine, the choice is a private remote or a deliberate
public push, and it is theirs.

Two local hooks make the default hold. Copy them from this skill's `hooks/` directory into
the repo's `.git/hooks` and `chmod +x` both. Copy rather than improvise: a
`reference-transaction` hook fires on every ref update, so a broken one wedges every git
operation in the repo.

- `pre-push` exits non-zero for `refs/heads/promotion`.
- `reference-transaction` refuses a deletion of that ref, which covers `git branch -D` and
  a forced `update-ref`.

Test both after installing: create and delete a scratch branch, try to delete `promotion`,
push a dry-run of `promotion` and of the default branch. Then say plainly that hooks are
per-clone and unversioned, so a fresh clone starts unprotected.

Neither survives `git clean -xdf`, which removes excluded files. Committed history does
survive it, in `.git`; only uncommitted edits and the worktree registration are lost, and
`git worktree repair` restores the registration. `git clean -xdf -e .promotion` avoids it.

## Recording history

Commit on the `promotion` branch at the end of every run, from `.promotion`, never from the
main working tree. One commit per run, with a subject naming what the run did:

```
git -C .promotion add -A
git -C .promotion commit -m "<what this run changed>"
```

Write the message the way any commit is written: what changed and why, not "update plan".
"Scope the native claim after finding softwaremill/realworld-spring-boot-native" is a
message worth reading in a year. Commit an amendment separately from the run that produced
it, so a later reader can tell a correction from a fresh assessment.

`## Log` and the git history answer different questions. The log is a dated record of what
the campaign did in the world, read inside the plan. The history is what the plan itself
said at each point, read with `git log -p`. Keep both.

## Schema

`.promotion/plan.md`. Everything except `## Constraints` and `## Baseline` is rewritten on
each run.

```markdown
## Constraints
Seeded once from intake question 6, then user-owned. Read it on every later run.

## Repo type
Primary type and reason, secondary or ruled out, and the channel shortlist with a reason
for every drop.

## Message
Target reader, their problem, the one claim, the evidence.

## Intake
The six answers, dated.

## Baseline
Dated snapshots, appended. Never replaced.

## Ceiling
Band, reasoning naming comparables and why they lead, tier-change condition, low-ceiling
verdict.

## Substance
| Action | Phase | Effort | Payoff | Confidence | Status |

## Audit
| Item | Score | Fix | Effort | Phase | Status |

## Distribution
| Action | Channel | One-shot | Phase | Effort | Payoff | Confidence | Metric | Status |

Phases are listed above the tables, each with its exit condition.

## Drafts
Links to files under .promotion/drafts/.

## Log
Dated entries: the approved set per run, then per action what ran, the resulting URL, and
the response.
```
