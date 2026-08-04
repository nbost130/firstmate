---
name: pr-shepherd
description: >-
  Thoroughly review one or more GitHub PRs and shepherd them to a MERGED
  outcome: inventory reviewer findings, verify each is fixed or intentionally
  deferred with evidence, drive CI to green, fix real failures, then merge.
  Success = merged PR. Use when the user asks to babysit/shepherd a PR, ensure
  reviewer comments are addressed, get CI green and land a PR, "make sure
  comments are fixed", "drive to merge", or invokes /pr-shepherd.
user-invocable: true
argument-hint: "<pr-number-or-url> [pr...] [--no-merge]"
metadata:
  internal: true
---

# pr-shepherd

`pr-shepherd` is a **gated PR landing pipeline** for *existing* open pull
requests. It is the counterpart of `no-mistakes` (which gates *your* uncommitted
work before/through open): you drive a fixed sequence of phases until the PR is
honestly landable, then **merge it**.

**Terminal success is a merged PR** (or a clear blocked report if gates fail).
Stopping at "merge-ready" without merging is only for `--no-merge` or a hard
authority boundary the user has not relaxed.

You are the AXI driver: every phase produces evidence; skip nothing; never claim
"comments addressed" without a per-finding disposition table.

## When to load

- Captain / user: `/pr-shepherd`, "shepherd this PR", "get comments fixed and CI
  green", "drive PR N to merge", "are reviewer comments addressed?"
- Firstmate: before reporting a ship PR ready, before autonomous merge under
  `yolo`, and any time a bot or human left formal or informal review feedback.
- **Standing captain preference (default):** shepherd ends with **merge when CI
  is green and reviewer comments are addressed** (gates 3–5). Do not wait for a
  second "ok to merge" unless the user passed `--no-merge` or a higher-priority
  hard boundary blocks merge (e.g. security-sensitive / red CI / open BLOCKER).

## Modes

| Invocation | Behavior |
|---|---|
| `/pr-shepherd <n>` or URL | Full pipeline; **merge** when gates 3–5 pass. |
| `/pr-shepherd <n> --no-merge` | Same pipeline; stop at merge-ready and report only. |
| Multiple PRs | Process **bottom-up** if stacked; otherwise one at a time. Stack desync is a hard gate. Merge each as it clears gates. |

Plain language "shepherd" / "drive to land" / "get it merged" means default merge
mode, not `--no-merge`.

## Hard rules

1. **Never invent "addressed."** Every unresolved thread, formal CHANGES_REQUESTED
   body, and consensus BLOCKER/WARN from bot review gets a disposition row.
2. **Never merge red CI.** Optional/non-blocking checks (e.g. long-running
   advisory review jobs you have evidence are non-required) may remain pending
   only when explicitly classified - see §CI.
3. **Merge is the default terminal step** after gates 3–5 pass. Use `--no-merge`
   only when the user asked to stop short. Still never merge red or with open
   BLOCKERs.
4. **Never force-push without `--force-with-lease`.** Prefer rebase + lease after
   base advances.
5. **Never discard unlanded work** to clear a path.
6. **Dead-code / wrong-surface findings are blockers.** If a review says the
   change is on an unused component, **verify importers with `git grep` / search**
   before disposing as NIT.
7. **Do not equate "I approved it" with "reviews addressed."** Your approval does
   not clear bot BLOCKERs or open threads.

## Pipeline overview

```
intake → inventory → thorough-review → comments → ci → base/stack → report → merge
```

Each phase ends with a **gate**. Fail-closed: missing evidence = not ready.
Success path always includes **merge** unless `--no-merge`.

---

## Phase 0 - Intake

1. Resolve owner/repo (default: `gh repo view --json nameWithOwner`).
2. For each PR number/URL:
   ```bash
   gh pr view <n> --repo <owner/repo> --json number,title,state,isDraft,url,baseRefName,headRefName,headRefOid,mergeable,mergeStateStatus,reviewDecision,author,commits,files
   ```
3. Refuse CLOSED/MERGED (report only). Undraft if the user wants land and the
   only draft reason was temporary hold **you** placed - otherwise ask.
4. Detect stack: walk open PRs where `baseRefName` is another PR's `headRefName`.
   If stacked, process bottom-up; if desynced (independent rebases of the same
   stack), stop and fix topology before comment/CI work (rebase onto base PR or
   main, thin the upper PR to unique delta).

**Gate 0:** Open PR identity, base/head SHAs, stack map recorded.

---

## Phase 1 - Inventory (read-only evidence pack)

Collect **all** of the following before editing anything:

### 1a. Human + formal reviews

```bash
gh api repos/<o>/<r>/pulls/<n>/reviews --jq '.[] | {user: .user.login, state: .state, submitted_at, body}'
```

Note DISMISSED vs active CHANGES_REQUESTED / APPROVED.

### 1b. Unresolved review threads (paginate)

GraphQL `reviewThreads` with `isResolved`, `isOutdated`, full comment bodies,
paths, authors. **Page until `hasNextPage` is false.**

### 1c. Bot / issue review comments

```bash
gh api repos/<o>/<r>/issues/<n>/comments --jq '.[] | select(.user.login|test("bot|github-actions|gemini|claude|copilot|coderabbit";"i")) | {user: .user.login, body}'
```

Parse **BLOCKER**, **WARN**, **NIT**, **CONSENSUS**, **CHANGES_REQUESTED** from
bodies (including `<!-- claude-pr-review -->` style).

### 1d. Checks

```bash
gh pr checks <n> --repo <o>/<r>
# and/or
gh pr view <n> --json statusCheckRollup
```

### 1e. Diff reality

```bash
gh pr diff <n> --name-only
gh pr diff <n>   # or compare main...head for content questions
```

For any finding that claims "unused" / "dead component" / "zero importers":

```bash
git fetch origin <headRefName>
git grep -n '<SymbolOrFilename>' origin/<headRefName> -- '*.ts' '*.tsx' '*.js' ...
```

**Gate 1:** Written inventory exists (in chat or a scratch note). No phase 2+
without it.

---

## Phase 2 - Thorough review (independent read)

Before trusting prior approvals, do a **fresh** pass on the current head:

1. Read the PR body for claimed scope and verification.
2. Walk the diff for correctness, security, product mismatch, and tests.
3. Cross-check bot/human findings against the code (confirm or refute with file
   evidence).
4. Flag new issues you find that nobody mentioned.

Optional: run a cross-vendor PR review skill if available; still validate every
cited path against the real diff (hallucinated paths do not count).

**Gate 2:** Short "shepherd review" note: approve-with-findings, or list new
blockers. Empty "LGTM" without inventory is invalid.

---

## Phase 3 - Comments gate (hard)

Build a disposition table for **every** item from Phase 1:

| ID | Source | Severity | Summary | Disposition | Evidence |
|----|--------|----------|---------|-------------|----------|
| t1 | thread | … | … | fixed \| reply \| defer \| wontfix \| outdated | commit SHA / reply URL / reason |
| b1 | bot BLOCKER | … | … | … | … |

### Disposition rules

| Severity | Allowed dispositions |
|----------|----------------------|
| **BLOCKER** / formal CHANGES_REQUESTED consensus | **fixed** (code + push) only. Reply after push with SHA. |
| **WARN** (consensus or product-correctness) | **fixed** preferred; **defer** only with user/captain OK and backlog note; **wontfix** only with written technical rebuttal posted on the thread. |
| **NIT** | fixed if cheap; else reply with rationale. |
| Outdated / already on main | **outdated** with SHA proof. |
| Question / clarification | **reply** with substantive technical answer (never "will fix" / "ack"). |

### Required code verification for "fixed"

1. Confirm the change is on the **PR head** (not only local dirty tree).
2. Re-read the resolved path on `origin/<head>` after push.
3. For dead-surface claims: re-run importer search post-fix.

### Cap and re-run

- Prefer fixing all WARNs that touch correctness in the same PR.
- After pushes, **re-fetch** reviews/threads/bot comments - old DISMISSED
  reviews do not prove the new head is clean; wait for re-review when a formal
  bot CHANGES_REQUESTED was active.

**Gate 3 (comments-addressed):** Zero open **BLOCKER** dispositions other than
`fixed` with evidence; zero unresolved threads that still need code; every WARN
either fixed or deferred with explicit authority. If any row is incomplete →
**not ready**.

---

## Phase 4 - CI gate (hard)

1. Classify checks:
   - **Critical:** anything that is required for merge, all `CI` / test / lint /
     typecheck / quality / build / guardrail jobs that normally block, and any
     check the repo treats as required.
   - **Advisory:** clearly non-blocking review bots (only if you have evidence
     they never block merge, e.g. optional check runs). Default: treat unknown
     as **critical**.
2. On failure:
   - Pull logs (`gh run view <id> --log-failed` or job URL).
   - Fix on the PR branch in an isolated worktree when under firstmate; push.
   - Re-wait.
3. Do **not** claim green while critical checks are `pending` or `fail`.
4. Flaky single flakes: one re-run max, then fix root cause.

**Gate 4 (ci-green):** All critical checks `pass` (or `success`); no critical
`fail`/`cancelled`/`timed_out` without resolution.

---

## Phase 5 - Base / stack / mergeability

1. `mergeable` / `mergeStateStatus`: resolve CONFLICTING with rebase onto
   current base (stack-aware: bottom first).
2. If behind main but clean, prefer update/rebase so CI matches landing base.
3. Stack: upper PR diff vs main should **not** re-introduce lower PR content
   after lower merges.

**Gate 5:** `MERGEABLE` (or known-platform lag with recheck); stack consistent.

---

## Phase 6 - Report (always, before merge)

Emit a captain/user-facing summary **in outcomes, not mechanics**:

```markdown
## PR shepherd: <title>
URL: https://github.com/.../pull/N

### Status: merging | blocked | merged

### Comments
| Finding | Disposition | Evidence |
| ... | ... | ... |

### CI
Critical: green | red (list failures)

### Shepherd review
1–5 bullets of independent findings (or "none beyond inventory")

### Next
- merging now / merged <url> / blocked on <decision>
```

If under firstmate, translate internal terms per AGENTS.md section 9.

**Gate 6:** Report prepared. Proceed to Phase 7 unless blocked or `--no-merge`.

---

## Phase 7 - Merge (default terminal success)

Merge when **all** of:

1. Gates 3–5 passed on the **current** head SHA (comments addressed, critical CI
   green, mergeable/stack ok).
2. User did **not** pass `--no-merge`.
3. No higher-priority hard boundary (destructive/irreversible beyond normal
   merge, red CI, open BLOCKER). Soft bot WARNs with disposition
   fixed/wontfix/defer-with-reason do not block.

Prefer the project's merge path:

- Firstmate: `bin/fm-pr-merge.sh <task-id> <url>` when a task owns the PR.
- Else: `gh pr merge <n> --squash` (or repo default). Use `--admin` only when
  code/CI gates passed and the only remaining block is a non-code review
  requirement the platform still shows (e.g. stale formal bot review that was
  fixed and re-verified).

After merge: confirm `state=MERGED`, full URL, one-line outcome. Firstmate:
fleet-sync clone, teardown only when unlanded-work checks pass.

**Done criteria for the skill:** `outcome: merged` with URL - not "checks green"
alone. Worker status lines should prefer
`done: PR <url> merged` after land; `checks green` is an intermediate claim only.

---

## Firstmate integration

When running as firstmate:

| Concern | Rule |
|---------|------|
| Project edits | Crewmate / isolated worktree only (hard rule 1). |
| PR open from ship | After worker reports green, **still run Phase 1–3** before merging. |
| Bot formal CHANGES_REQUESTED | Block merge until re-run clears or code fixes land. |
| Status line `done: PR … checks green` | Treat as worker claim; **re-verify** gates 3–4 yourself, then merge. |
| Shepherd terminal | Default **merge** when gates pass (captain standing preference). |
| `--no-merge` | Only when captain asked for report-only. |

Suggested captain invocation: `/pr-shepherd 4125` or
`/pr-shepherd https://github.com/org/repo/pull/4125`.

---

## Relationship to other tools

| Tool | Role |
|------|------|
| `no-mistakes` | Pre-merge validation of **your** branch/pipeline work. |
| `pr-babysit` | Multi-PR loop with auto-fix; **never merges**; use for watch lists. |
| **pr-shepherd** | **Thorough single/stack land gate** ending in **merged PR** (unless `--no-merge`). |

Prefer `pr-shepherd` when honesty of "comments addressed + CI green + landed"
matters. Use `pr-babysit` for ongoing multi-PR watch without merge authority.

---

## Anti-patterns (learned the hard way)

- Approving and merging while a bot **BLOCKER** is still formal CHANGES_REQUESTED.
- Wiring UI to a component with **zero importers** because the PR body said so.
- Treating quality/knip flake as "whole tree debt" without running the ratchet.
- Claiming CI green when only title/body lint ran (full CI never triggered).
- Merging the stack top while it still contains the bottom's files after both
  rebased independently.
- Leaving collapsed a11y WARNs unfixed on the PR that introduced them.

---

## Minimal command cheatsheet

```bash
# Identity + checks
gh pr view N --json url,state,headRefOid,reviewDecision,mergeable,statusCheckRollup
gh pr checks N

# Reviews + threads (see Phase 1 for full GraphQL pagination)
gh api repos/O/R/pulls/N/reviews
gh api repos/O/R/issues/N/comments

# Diff + importers
gh pr diff N --name-only
git grep -n Symbol origin/branch -- '*.ts' '*.tsx'

# After fix
git push --force-with-lease   # only if rebase rewrote
gh pr comment N --body "..."

# Merge (only when authorized + gates green)
gh pr merge N --squash
```
