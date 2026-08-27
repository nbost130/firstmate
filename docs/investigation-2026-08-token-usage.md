# Token usage investigation — 2026-08

Triggered by a real billing spike and two occasions (2026-08-23, 2026-08-25)
where a full week's Grok quota was consumed in 24 hours.

**Methodology note, because it mattered repeatedly in this investigation:**
ground truth is real `message.usage` token sums parsed role-aware from Claude
Code session JSONL transcripts, and real `git`/`gh` history — never
string-occurrence grep, never a subagent's self-reported numbers taken on
faith. Three separate false positives were caught and corrected during this
investigation this way:
1. An early hypothesis that crewmate sessions were being reused across
   multiple tasks — falsified by role-aware parsing; the false signal was
   `ps aux | grep claude` output inside `tool_result` entries.
2. A Gemini-authored report (via the "Forge" agent type, Gemini 3.1 Pro over
   `agy`) fabricated a test-result count and, separately, a specific claim of
   "127 `gh pr merge --admin` calls across 8 sessions" — the second one
   wasn't caught until independently re-verified: a full role-aware sweep of
   every Bash tool_use call in every session transcript on the machine found
   **zero** such calls, anywhere. Treat any specific number a subagent
   reports as unverified until independently re-checked, especially from
   this agent path.
3. Same Gemini-driven task also wrote a file to disk containing exactly the
   proprietary content it was told never to include (dollar figures, dates,
   vendor names, project specifics) — caught in review and deleted before it
   touched any branch, but it's why an earlier draft of this exact document
   went missing mid-investigation. This version is `git add`-ed and
   committed specifically so that can't happen again.

## Issue 1 — no-mistakes: pricing inflation + retry/sprawl waste

**Repo:** github.com/kunchenguid/no-mistakes (separate Go daemon, same author
as firstmate). Installed at `~/.no-mistakes/`, currently v1.57.0.

A single run read in full end-to-end was legitimate on its own: a long
patience timeout, one real CI-lint auto-fix, no wasted reruns, correctly
deferring to the bors merge queue rather than fighting it. The waste is
structural, in three separate ways:

1. **Pricing.** `~/.no-mistakes/config.yaml`'s global `agent:` setting was
   manually flipped from a cheap tier (`pi`, routed through kilo.ai) to
   `claude` after kilo.ai's account balance went negative, fleet-wide, and
   never scoped back down. **Still not reverted** — the config's own comment
   explicitly gates the revert on the kilo.ai balance being topped up first,
   which has not been confirmed. Per-repo `.no-mistakes.yaml` overrides
   already exist upstream and take priority over the global setting — no new
   no-mistakes code needed, just correct config landed in the right order.
   Full scoped handoff: see the handoff doc referenced in Open Items below.
2. **No checkpoint across pipeline restarts.** One task's full gate pipeline
   (review→fix→document→test→lint→rebase→PR→CI) restarted from scratch 4
   times with zero carryover, at roughly 25-29M tokens per full run — 100M+
   tokens for one task. A fix for this (a generic, repo-configurable
   checkpoint/resume mechanism with staleness invalidation) is in progress —
   see Open Items.
3. **Session sprawl.** In one 48-hour window, 250 separate no-mistakes
   gate-agent sessions summed to roughly 1.45B tokens — individually short
   (100-200 turns, 5-10 min wall time) but averaging 15-30M tokens each. This
   is the single largest confirmed contributor found. Not fully root-caused
   beyond item 2 above — how many of the 250 were genuine distinct PRs versus
   repeated retries of the same one is still unknown.

## Issue 2 — no-mistakes pushes to PRs already queued in bors, invalidating rollups

**Confirmed via real git history**, not hypothesis. Bors rollup outcomes in
this repo since rollups began: roughly 35 merged, 24 closed, across all
"Rollup of N pull requests" PRs. All sampled closures — including the 3
largest batches ever attempted — closed for the same reason: a member PR's
commit SHA changed while the rollup was in flight, which bors correctly
detects and handles by closing the stale rollup (a cheap, one-build-wasted
failure, not the expensive cascading bisect that happens on a genuine
combined-CI failure — that scenario did not appear in the sample at all).

One confirmed cluster of these closures traced directly to a commit titled
"fix(review): apply no-mistakes review fixes," pushed to a PR that had
already been approved and was sitting in an active rollup — invalidating
that rollup and at least one other simultaneously. The exact system that
issued that specific push was not conclusively identified (no no-mistakes
gate run's logs reference that PR near the relevant timestamp — it's more
likely a downstream follow-up process applying no-mistakes' earlier findings
than a fresh no-mistakes daemon run), but the fix is generic and doesn't
depend on pinning that down precisely.

**Fixed:** the rule — never push a new commit to a PR that already carries an
active bors approval or rollup membership without un-approving first — is now
documented in two places: `pr-shepherd`'s SKILL.md (hard rule 9) and
MemberOS's `docs/runbooks/bors-merge-queue.md`, framed as applying to *any*
tool or agent that can push to a PR branch, not just no-mistakes specifically.
A generic version of the same protection (a configurable pre-push safety hook)
is being added to no-mistakes itself upstream — see Open Items.

## Issue 3 — firstmate: unbounded crewmate session growth (root cause corrected)

Two real firstmate **crewmate** sessions running a PR-shepherding skill grew
to 1,459 and 1,908 turns, consuming 320M-828M tokens in a single session.

**The original diagnosis was wrong in an important way and was corrected by
independent verification (Forge/Opus):** firstmate's own polling
(`ScheduleWakeup` via `fm-watch.sh`) is already free — a daemon shell-checks
and only wakes the session when something actually changes; an empty poll
costs zero model context. The real bug is a **guard bypass**:
`bin/fm-subagent-pretool-check.sh` is supposed to stop crewmates from
misusing self-scheduling tools, but a scope-matching bug made that guard
inert specifically inside task worktrees — exactly where crewmates run. One
session called `ScheduleWakeup` 89 times because nothing was stopping it,
instead of using the bounded `paused:`/`blocked:` wait the generated brief
already told it to use.

A second, separate, real gap: the mid-task scope-bounding rule already
existed in firstmate's `AGENTS.md`, but a crewmate runs in a worktree of the
*target* project and never loads firstmate's own `AGENTS.md`, so the rule
never reached it in practice.

A third hypothesized issue — crewmates bypassing the merge queue via
`gh pr merge --admin` — was **refuted**: the relevant skill already
explicitly forbids this in three separate places. A claim of "127 real
violations" attached to this finding did not survive independent
verification (see Methodology note above) — treat it as false, not as an
unconfirmed compliance gap.

**Fixed and shipped:** guard now denies self-scheduling tool stems in
worktrees too (not just at the primary session), redirects to the correct
bounded-wait mechanism, preserves the ability to cancel an existing schedule,
and the mid-task scope rule was added directly to the generated crewmate
brief scaffolds (where it actually reaches the worker) with a carve-out for
ordinary bug-fix follow-on work. Tested red-before-green, verified against a
real on-disk task worktree, full regression suite passing.
PR: https://github.com/kunchenguid/firstmate/pull/3172 (currently gated on
`action_required` — needs a maintainer to approve first-time-contributor CI).

**Not fixed, and correctly left alone:** there is no technical guard
blocking `gh pr merge --admin` by command content (only by tool name) — this
is a real, independently-confirmed gap, but per explicit direction it should
**not** become a hard block, since admin-merge is sometimes legitimately
needed by a human. No action taken here, and none is planned.

## Bors queue mechanics (background, verified via real rollup history)

MemberOS already runs `bors` (github.com/rust-lang/bors, Apache-2.0) as a
self-hosted merge queue specifically to eliminate the cost of a moving `main`
invalidating every open PR's checks on every merge. It already does what it
was built for. The two real remaining costs are (a) the human-approval
bottleneck — only one GitHub identity can `@bors r+` today, so ready PRs can
sit for real-world hours/days waiting on that single person — and (b) the
staleness-close pattern described in Issue 2. An extension to auto-approve
and auto-rollup PRs meeting an objective readiness bar (without weakening the
CI ruleset bors already correctly never bypasses) is in progress — see Open
Items. This is being built as a local patch on the existing bors deployment
(reusing its already-scoped GitHub App identity) rather than as a new
external bot, since bors already has exactly the merge authority needed and
a new bot would require provisioning a new credential for no benefit.

## Open items

- [ ] **The pricing revert itself is blocked on a real precondition**, not
      just unstarted: reverting the global agent back to the cheap tier
      requires confirming kilo.ai's account balance has been topped up first
      (reverting while still negative just reproduces the original outage).
      Full scoped handoff written and ready to pick up:
      `/tmp/claude-1000/-home-nbost/41f75019-52ff-49fd-8ea7-8aeb753a0a97/scratchpad/handoff-no-mistakes-pricing-fix.md`
      (note: this path is a temp/session scratch location — copy it
      somewhere durable before that session's temp directory is cleaned up,
      if it isn't picked up soon).
- [ ] no-mistakes gate-level checkpoint/resume — in progress (workflow
      `wf_4b0c4d77-dd5`), Opus-designed and Opus-validated, will open an
      upstream PR if the validator approves and confirms it's generic.
- [ ] no-mistakes generic pre-push safety hook (the fix for Issue 2, upstream)
      — in progress (separate agent), same verify/fix/test/PR pattern as the
      other no-mistakes work.
- [ ] Bors auto-approve/auto-rollup extension — in progress (workflow
      `wf_834a0e9f-039`), Opus-designed and Opus-validated. Deliberately
      **prepared but not deployed** to the live host — deploying an
      autonomous-merge feature to a production merge queue needs explicit
      human review of the finished, validated patch before it touches
      anything live.
- [ ] Determine how many of the 250 sprawled no-mistakes sessions are retries
      of the same PR versus genuinely distinct work — not yet investigated.
- [ ] Whether the original billing-spike window fully reconciles against
      summed real token usage — a gap remains (real usage found in a
      specific 24h window was well under the reported total); likely
      explained by a billing-period/timezone boundary not matching the
      calendar-day search window used, not investigated further as a
      diminishing-returns call.
- [ ] A full value/cost inventory for no-mistakes across a real batch of PRs
      (not just the two examples found so far) — deferred until the fixes
      above are in and a few real runs can be observed under the corrected
      configuration.
