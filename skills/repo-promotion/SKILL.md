---
name: repo-promotion
description: Plan and run a promotion campaign for a GitHub repo. Use when the user wants more stars, wants a project noticed, asks how to promote or launch a repo, or asks why a repo isn't getting traction.
---

# Repo promotion

Plan first. Execute after the user confirms. Planning writes only `.promotion/`, which the
repo excludes, so no tracked file changes before the user has seen the plan.

Three kinds of work carry a repo: **substance** makes it worth starring, **conversion**
turns a visitor on the page into a star, **distribution** brings the visitor. Conversion
gates distribution, because every link posted anywhere lands on the page as it stands that
day.

Some channels are **one-shot**. A repo gets one Hacker News submission and one subreddit
front page that matter. Spending one before the page converts throws it away. Order the
plan so one-shot channels go last.

## Guardrails

- Every tactic in the plan must be one the user would be happy to see attributed to them
  publicly, by name, in the repo's own community. That standard admits real distribution
  and excludes bought stars, star-for-star rings, mass DMs, and comment spam.
- State attribution only where referrer or timing data supports it. Where it does not,
  write "unclear" and say why. Star counts under a few hundred are noisy and a false
  attribution sends the user back to a channel that did nothing.
- The plan stays out of everything the work produces. Commit messages, PR titles and
  bodies, issue text, posts, and list submissions carry the technical reason for the
  change and nothing about the campaign behind it. An audit item number, a ceiling
  estimate, a note that a repo is being promoted at all: none of that belongs in a public
  artifact, and every one of those artifacts is permanent. State why the change is right
  on its own terms, which is a better argument anyway.
- A **low ceiling** ends the run at substance and conversion. Say so and stop, rather than
  producing a campaign the repo cannot win. Low means the band's upper bound sits under
  twice the current star count, or the repo type has no realistic star path at all.

## Step 1: resolve the repo and the mode

Prefer a local checkout, and confirm it is the right one:
`git -C <path> remote get-url origin` has to resolve to the repo being planned for. For a
remote `owner/repo`, clone shallow into a temp directory outside the checkout.

Check push access:

```
gh api repos/{owner}/{repo} --jq '.permissions.push'
```

Push access runs **full mode**: intake, plan, execute. No push access runs **reduced
mode**: audit and ceiling only, no intake, no execution, output is an assessment rather
than a campaign. Traffic data needs push access too.

Done when: the checkout's remote matches the target repo, the permissions call returned a
value rather than an error, and the mode is stated to the user in one line.

## Step 2: load state

Read `.promotion/plan.md` if it exists.

`## Constraints` is user-owned and binding. Drop any plan item that violates it.

On a first run the file does not exist yet. Create it and seed `## Constraints` from
intake question 6. After that first write it belongs to the user, so read it and leave it
alone.

With intake answers already in the file, restate them in one line and ask only whether
anything changed. Skip step 3.

Done when: constraints are known, and intake answers are either in hand or queued for step
3.

## Step 3: intake

Full mode, first run only. Ask all six in one message, then wait.

1. How many hours per week will you spend on this, sustained rather than a one-week burst?
2. What channels do you already have (blog, X, Mastodon, LinkedIn, talks, newsletter,
   employer reach), and roughly what reach on each?
3. Will you write long-form, record video, both, or neither?
4. What is the repo for: showing your work, attracting contributors, attracting users,
   teaching, or your own use? (These map onto the repo types in `CHANNELS.md`.)
5. Is there a deadline or event driving this (job hunt, conference, launch, funding)?
6. Anything off limits (channels, platforms, tone, time of day)?

Answers have to be usable. "A few hours" is not a weekly budget and "some LinkedIn reach"
is not a follower count. Ask again for anything that steps 8 and 9 cannot compute against.

Done when: all six carry an answer specific enough to plan against, with question 1 a
number of hours and question 2 a number per channel or an explicit none.

## Step 4: snapshot and diff

```
gh api repos/{owner}/{repo} --jq '{stars:.stargazers_count,forks:.forks_count,watchers:.subscribers_count,issues:.open_issues_count}'
```

With push access, add traffic (14-day window, no history beyond that):

```
gh api repos/{owner}/{repo}/traffic/views
gh api repos/{owner}/{repo}/traffic/clones
gh api repos/{owner}/{repo}/traffic/popular/referrers
gh api repos/{owner}/{repo}/traffic/popular/paths
```

Under a few thousand stars, per-star timestamps are cheap and show when growth actually
happened:

```
gh api repos/{owner}/{repo}/stargazers -H "Accept: application/vnd.github.star+json" --paginate --jq '.[].starred_at'
```

Append the snapshot to `## Baseline` with today's date. Never overwrite an earlier one.

Stars, forks, watchers, and issues are cumulative, so they diff across runs. Traffic is a
rolling 14-day window, so a snapshot from three weeks ago describes a different fortnight
and does not diff. Read traffic as current state only.

On a re-run: diff the cumulative fields against the previous snapshot, mark every item in
`## Substance`, `## Audit`, and `## Distribution` done, skipped, or failed, and attribute
the delta to channels where the data supports it.

A re-run has to change the next plan, otherwise it is bookkeeping. Drop a repeatable
channel that produced no measurable traffic across two attempts. Move conversion work back
to phase 1 when a channel delivered visitors and stars did not follow. Hold the next
one-shot when the audit has regressed.

Done when: all four cumulative fields are recorded, traffic is recorded or its absence
explained, every prior item across all three tables carries a status with evidence, and
the re-run states which of the three decisions above it took.

## Step 5: classify

Type the repo using the list at the top of [`CHANNELS.md`](CHANNELS.md). The type selects
which channels are in play at all.

A repo can legitimately be two types at once. Name a primary and, where one applies, a
secondary. Channels are the union of both, priority follows the primary.

Then build the **shortlist**: walk every channel entry in `CHANNELS.md`, keep the ones
whose repo-type fit matches, and drop the rest with a one-line reason. The shortlist is
what step 7 researches, so a channel missing here never gets considered again.

Done when: the primary type is named with a one-sentence reason, any secondary is named or
explicitly ruled out, and every channel entry in `CHANNELS.md` is either on the shortlist
or dropped with a reason.

## Step 6: audit

Work through every item in [`AUDIT.md`](AUDIT.md). Score each **present**, **weak**, or
**missing**, and write the specific fix for anything not present.

Some items carry an applicability condition. Where the condition is not met, score the
item `n/a` and name the condition. That counts as scored.

An item that bundles several checks scores to its weakest component, and the fix names
that component. Scoring the easy half and moving on is the failure this rule exists to
catch.

Done when: every item in the file has a score or an `n/a` with its reason, and every weak
or missing item has a fix a person could execute without asking a follow-up question.

## Step 7: research

Delegate the comparables hunt to a subagent where the host allows it, since it is a wide
search whose raw results are noise. Read channel rules and list requirements yourself. A
conclusion relayed by a subagent is a recollection, and the criteria below require having
read the source.

Three targets:

- **Comparables**: 5 to 10 repos serving the same need at similar age and maintenance
  level, each with its star count and a reason it has them. The reason has to cite
  something observable: org ownership, the date it joined the canonical index, the
  author's follower count, package download counts, a linked article or book, a visible
  launch spike in its star history. Where nothing observable explains the gap, say so and
  mark the reason inferred.
- **Channel rules**: current posting rules for every channel on the shortlist.
  Self-promotion policies change and community tolerance changes faster, so read the rules
  rather than recalling them.
- **List targets**: the curated lists worth submitting to, named by URL, with each
  contribution requirement read and checked against this repo. Check whether the repo
  already holds a placement before planning to win it. Search GitHub topics for the
  ecosystem, search `awesome <ecosystem>` and `awesome <language>`, and find the canonical
  index for the repo's type. Stop when two consecutive searches surface nothing new.

Some sources block reading. When the rules for a channel cannot be reached after a real
attempt, mark that channel **rules unread**, say what was tried, and carry any item for it
as conditional on the user checking the rules first. Guessing at a community's current
self-promotion policy is how accounts get banned.

Done when: comparables carry counts and reasons, every shortlisted channel has its current
rules recorded or is marked rules unread with the attempts listed, and every list target
is a URL with a checked requirement rather than a category.

## Step 8: ceiling

Place the repo in the comparables' star distribution. State a band, the reasoning, and
what would have to change to move up a tier.

A band with a mechanism is a claim the user can check. State a band, never a projected
number. Keep the upper bound within three times the lower bound, over a stated horizon: a
band wide enough to be always right says nothing.

Ask why the comparables are ahead, because the answer changes the plan. A repo behind on
quality needs substance work. A repo behind on time in an index needs distribution and
patience. These look identical in the star count and call for opposite plans.

Then apply the low-ceiling test from the guardrails and state the verdict either way.

Done when: the ceiling is a band no wider than three times its lower bound over a stated
horizon, the reasoning names comparables and cites observable evidence for why they lead,
the tier-change condition is a concrete event rather than an aspiration, and the
low-ceiling verdict is stated.

## Step 9: build the plan

Reduced mode and a low-ceiling verdict both stop here. Write the audit, the ceiling, and a
reframing proposal instead: what this repo would have to become, or what to extract from
it, to be worth promoting. Skip `## Distribution` entirely and go to step 10.

Otherwise, three sections. Substance and distribution items carry effort, expected payoff,
and a confidence mark. Audit items carry their score and fix from step 6, an effort
estimate, and their phase from the dependency rule below. They take no payoff or
confidence, because a conversion fix is gating rather than ranked, but their effort counts
toward the budget like any other work.

- `## Substance`: product work that makes the repo more worth starring. Extracting a
  reusable piece, the one capability no comparable has, a template repo, a genuinely novel
  claim. Often the highest payoff and usually the highest effort.
- `## Audit`: the fixes from step 6. Cheap, permanent, and they improve every future link.
- `## Distribution`: channel by channel, each target named specifically. Check the owner's
  other repos too: a sibling project in the same niche is cheap cross-linking that no
  external channel can match.

Effort is S=1, M=3, L=8, in hours. Payoff is High=8, Med=3, Low=1. Order by payoff divided
by effort. The numbers are a tie-breaker for judgment, not a replacement for it, but they
make the ordering something the user can argue with.

Then check the plan against the intake budget. Sum the effort hours per phase, audit rows
included, and divide
by the weekly hours from intake question 1. A phase that runs past a few months at that
rate is too big, so cut its lowest-ratio items until it fits and say what was cut. A
budget collected and then ignored produces a plan the user silently abandons.

Group into phases so dependencies hold: fixes first, permanent placements (curated lists,
registry pages, ecosystem surfaces) second, one-shot channels last. Every table carries a
`Phase` column, so audit fixes sit in phase 1 in their own table rather than being
represented by a placeholder elsewhere. Phases, not dates. Dates against a self-reported
hours-per-week budget go stale in a week and make the plan read as failed.

Every phase states its exit condition, which is what makes phases work where dates do not.
Phase 1 exits when the audit items blocking conversion are verified fixed and the demo
responds. A one-shot phase cannot start until the phase before it has exited.

"Post to a relevant subreddit" is not a plan item. "r/java, self-promotion allowed under
the current rules only alongside technical content, link the write-up rather than the
repo" is.

Write `## Message` before the tables: who the target reader is, the problem they have, the
one claim this repo makes to them, and the evidence for it. Every draft and every channel
pitch says the same thing, and this is where that thing is decided.

Write the whole thing to `.promotion/plan.md` using the schema in [`STATE.md`](STATE.md),
which also covers where that directory lives and how its history is kept.

Done when: every substance and distribution item carries effort, payoff, and confidence;
every distribution item names a specific target and a metric to judge it by; every phase
has an exit condition; the summed effort fits the intake budget or the cuts are stated;
`## Message` is filled; the plan file matches the schema.

## Step 10: present and gate

Show the plan and stop. Nothing beyond the plan file executes before the user confirms.

Record what the user approved, item by item, in `## Log`. That approved set is the only
thing step 11 may act on, so an unrecorded approval is no approval.

Done when: the plan has been shown, the user has answered, and the approved set is written
to `## Log` with the date. A low-ceiling or reduced-mode run ends here.

## Step 11: execute

In-repo edits run as one batch into the working tree. No commit, no branch. Report what
was touched and leave the commit to the user.

Anything that leaves the repo gets confirmed on its own, every time, including
awesome-list PRs opened through `gh` under the user's account.

Posts for aggregators, forums, and social go to `.promotion/drafts/<channel>.md`, one file
per post, each carrying the target, the rules that apply, and the body. Write every draft
in first person as the repo author, and apply the `unslop` skill before showing it. A post
that reads as generated gets called out, and that becomes the thread.

Commit the plan when the run ends, per [`STATE.md`](STATE.md). A run that changes the plan
and leaves it uncommitted loses the one record of what changed and why.

Done when: every item in the approved set is either executed or carries a reason naming
what blocked it, `## Log` records for each one the action, the date, the resulting URL
where there is one, and what the target's response was, and the plan is committed.

## Plan file state

[`STATE.md`](STATE.md) holds the schema, the directory layout, and the history rules. Read
it before the first write of a run and before the last.
