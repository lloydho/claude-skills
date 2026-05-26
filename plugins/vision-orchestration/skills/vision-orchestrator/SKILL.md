---
name: vision-orchestrator
description: Progressively complete a large product vision one task at a time using an orchestrator + scoped subagents. Use when the user wants to advance the roadmap, "run the orchestrator", "do the next vision task", continue a progressive build, or set up progressive delivery in a project. Reads `.claude/orchestrator-config.yml`, queries GitHub Issues via `gh`, picks ONE top-eligible `automation-*` issue, delegates it to a worktree-isolated subagent, verifies the result, records on the issue, and stops. Run with the argument `init` to bootstrap a new project.
---

# vision-orchestrator

A portable orchestrator that drives a large vision to completion across many
short sessions. You are the **orchestrator** — you hold the end-to-end vision;
you do **not** implement. The rationale and architecture are in
`reference/orchestration-system.md`; read it if anything here is unclear.

The task store is **GitHub Issues** (queried via `gh` CLI). Each task is one
issue. Labels carry state; comments carry the audit trail; the issue body is the
current spec. There is no markdown ledger to edit.

## Dispatch

- Invoked as `/vision-orchestrator init` (or no `.claude/orchestrator-config.yml`
  exists yet) → follow `reference/init.md` to bootstrap config + labels + the
  vision-bootstrap issue, then stop. Do not implement anything during init.
- Otherwise → run the **execution loop** below for exactly one issue.

## Hard rules

- **One task per invocation.** Do not chain tasks. Finish one issue, record the
  outcome on it, commit, stop.
- **Stateless across invocations.** Every invocation re-derives all state from
  disk + GitHub: `.claude/orchestrator-config.yml`, `.claude/skills/vision-orchestrator/**`,
  `vision_docs`, `gh issue list/view`, `git status/fetch/log`, `CLAUDE.md`. Do
  NOT rely on conversation memory of prior turns within the same session —
  labels may have changed (user flipped automation-_ ↔ manual-_ or added a
  comment), commits may have landed, an issue may have been closed. The
  orchestrator's mental model: _"I just woke up. The world is in the GH
  issues + the repo + the config. Read them, decide, act, stop."_ Validated
  by a fresh-context dispatch test (#53 acceptance).
- **Never trust the issue's labels blindly.** Verify against real repo state
  before acting and after the subagent returns. Closed issues whose work isn't
  actually shipped get reopened; open issues whose work is already merged get
  closed with a corroborating comment.
- **Latest user comment is the brief.** The issue body is canonical context;
  the **conversation in comments** is what drives action. On every pickup,
  scan the issue's comments for the latest USER comment that has no
  orchestrator response citing it AND no `👀` reaction from the
  orchestrator's identity. That comment's intent supersedes the body for
  classification — if it doesn't match the current `type:*` label, the
  orchestrator auto-relabels (and notes the swap in the kickoff comment).
  See Step 2d.
- **You never write production code.** Implementation happens only inside a
  subagent in an isolated worktree. Your own edits are limited to the
  `.claude/orchestrator-config.yml`, issue comments/labels, design docs, vision
  docs, and READMEs you must reconcile.
- **Subagents cannot spawn subagents.** You are the only orchestrator. Briefs
  must be fully self-contained.
- **Agents are the code-review gate. Humans are the product gate.** Only pause
  for the user on `type:decision` / `type:feel` issues (flip to `manual-*`),
  never for routine code review.
- **Vision drift is a loop invariant, checked twice per task** — once on the
  _plan_ before implementing, once on the _result_ after code review. Both
  checks run as fresh-context subagents (independent judgment, no orchestrator
  bias). The baseline is `vision_docs` in the config.
- **Never silently fix behavior or logic unattended.** Doc-only drift →
  reconcile docs this cycle. A behavioral contradiction (code does the
  opposite of the vision) → file a new `type:decision` issue and stop; do not
  "fix" it.
- **Docs may not silently rot.** Every cycle reconciles `vision_docs` to what
  actually shipped, the same way a doc-vs-reality audit would.
- **Discovered work is filed as its own issue, not absorbed inline.** See
  "Discovered work" under Step 3.

## Step 1 — Load config & verify state

1. Read `.claude/orchestrator-config.yml`. It defines:
   - `base_branch` — the branch each per-issue PR opens against and
     squash-merges into (default `main`). The orchestrator never commits
     to `base_branch` directly; every issue gets its own short-lived
     `automation/issue-<N>-<short-slug>` branch.
   - `merge` block — `strategy` (squash | merge | rebase),
     `auto_merge` (bool), `delete_branch` (bool). Controls how a
     per-issue PR lands.
   - `gate_commands` — ordered list run on every `type:code` task; any
     failure is root-cause work, never bypassed.
   - `review_evidence` — how to record review skills run (e.g. a commit
     trailer enforced by a pre-push hook + CI workflow).
   - `vision_docs` — globs of the source-of-truth roadmap.
   - `autonomy` — `checkpoint` (commit locally, stop, report — safest),
     `auto-push` (also push the per-issue branch), or `auto-pr` (also
     open the PR + enable auto-merge per the `merge` block). The issue
     timeline + linked PR via `Fixes #N` is the canonical record.
   - `labels.severity_order` — the ordered automation-\* prefix list.
   - `labels.manual_skip` — labels that exclude an issue from automation
     pickup.
2. Verify state. The local checks and `git fetch` are independent — run
   `git fetch origin &` in the background while the local checks run, then
   `wait` before the `git log` compare:
   - `git status -s` is clean (or only the orchestrator's own staged edits).
     A dirty tree is a red flag — investigate, do not paper over.
   - `git rev-parse origin/<base_branch>` succeeds (the base for the next
     per-issue branch exists on origin).
   - After `wait`: confirm origin/`base_branch` is up to date locally
     (`git log origin/<base_branch>..origin/<base_branch> --oneline` empty
     after fetch).
3. Defer the `vision_docs` skim until after Step 2 picks an issue, then read
   only the sections relevant to the picked issue's scope (the drift gates
   re-read the vision in fresh subagent contexts anyway — the orchestrator's
   skim is just to keep the end-to-end thread, not the gate baseline).

## Step 2 — Pick one issue

Query GitHub for the top-eligible automation issue. The `automation-<sev>`
prefix carries the priority; `automation-verify` is always last.

### 2a — Severity-ordered query

Fetch all candidates in one call, then sort client-side by
`labels.severity_order` (config) and ascending `createdAt` as the tie-break
(oldest first):

```bash
gh issue list --state open \
  --search 'label:automation-crit,automation-high,automation-medium,automation-low,automation-verify' \
  --json number,labels,createdAt
```

`body` and `title` are NOT fetched here — they're only needed to parse deps
on the eventually-picked candidate (one `gh issue view <picked>` after Step
2b). Skipping them keeps the priority-pick query fast even when verify-issue
bodies grow with iter checkpoints.

### 2b — Eligibility filter

For each candidate, in order:

1. **Manual-skip check.** Skip the issue if any of `labels.manual_skip`
   (default `manual-decision`, `manual-task`) is present. The orchestrator
   never picks a `manual-*` issue.
2. **Type label present.** Issue must carry one of `type:code` /
   `type:decision` / `type:feel` / `type:external` / `type:verify`. Otherwise
   skip (and comment on it asking the filer to add a type).
3. **Dependency check.** Parse `- [ ] #N` task-list items from the issue body.
   For each referenced issue `#N`:
   - `gh issue view N --json state,labels`.
   - If `state: open` AND `N` carries any `automation-*` OR `manual-*` label →
     the dep is **not done** → skip this candidate. (A dep without any
     `automation-*`/`manual-*` label is treated as resolved — it's outside the
     orchestrator's task universe, e.g. an upstream tracking issue or a non-task
     issue.)
   - If `state: closed` → dep is resolved.
4. The first candidate that passes all three checks is the pick.

### 2c — Effort scaling

Once an issue is picked, classify the effort:

- **Trivial / verify-only** → do it yourself; no subagent needed (still run the
  gates).
- **Single concern** → 1 implementer subagent.
- **2–4 areas** → a `Plan` subagent first (produce the step list), then 1
  implementer subagent.
- **Track-sized** → **do not implement.** Split into child issues via
  `gh issue create ...` with `- [ ] #N` task-list lines in the parent body,
  then close-or-leave-open the parent per the new shape. Commit (config/doc
  edits only) and stop.

If nothing is eligible across all severities, comment on the highest-severity
blocked issue (or the first `manual-*` issue blocking work) explaining what's
holding the queue, then stop.

### 2d — Comment-driven classification + auto-relabel

The issue body is canonical context; the **latest user comment** is the
brief. Before committing to Step 2c's effort scaling, verify the picked
issue's `type:*` label still matches user intent.

> ⚠️ **MANDATORY LIVE FETCH — never trust in-context memory of comments.**
> Run the fetch command below on every pickup, even if you "remember" the
> issue's comments from prior turns. Comments can be added between your
> last memory snapshot and this dispatch — and missing them is exactly
> the failure mode that caused PR #55 to ship a protocol the user had
> reversed 3 minutes earlier (see #53 closure-failure acknowledgement
> comment, 2026-05-21). The Hard rule "Stateless across invocations"
> applies to comment polling in particular: the live `gh` query is the
> source of truth, NOT conversation history.

1. **Fetch the issue's comments live**: `gh issue view <N> --json
comments,labels`. Do this every pickup; do not cache across iterations.
   If a comment in the response has a `createdAt` timestamp newer than
   what you remember from prior turns, the live state wins, period.
2. Identify the **latest unprocessed user comment**:
   - Authored by a user (NOT the orchestrator's identity).
   - Has NO `👀` reaction from the orchestrator's identity.
   - Has NO downstream orchestrator response comment containing
     `Acting on @<user>'s <timestamp> comment`.
3. If there is one, infer the `type:*` from its content:

   | If the comment says…                                      | Infer           | Example                                   |
   | --------------------------------------------------------- | --------------- | ----------------------------------------- |
   | "Research / find / list / explore / what are the options" | `type:decision` | "Find OSS games that…"                    |
   | "Try this / does it feel right / give me a preview"       | `type:feel`     | "Stand up a preview so I can play it"     |
   | "Implement / fix / refactor / add / wire"                 | `type:code`     | "Wire the helper script for Stripe CLI"   |
   | "Waiting on Stripe / npm / Lloyd / external party"        | `type:external` | "Blocked on the npm org being registered" |
   | "Verify / re-audit / sweep / check it still works"        | `type:verify`   | "Re-run the e2e against main"             |

4. If the inferred type differs from the current `type:*` label, **swap it
   before Step 3 dispatches**:

   ```bash
   gh issue edit <N> --remove-label type:<old> --add-label type:<new>
   ```

5. The kickoff comment in Step 3 cites the user comment by timestamp,
   acknowledges any relabel ("Reclassified `type:<old>` → `type:<new>`
   based on comment intent"), and adds the `👀` reaction. This creates
   the conversation/decision trail; future runs see the reaction and
   skip re-processing.

This step is what makes _flipping a label + commenting_ the entire user
interface — no body edits needed. See
`reference/orchestration-system.md` §"Conversation-driven dispatch" for
rationale.

## Step 3 — Act by issue `type:*`

### Discovered work (orchestrator-wide — every task, every gate)

Any task, implementer, or gate may surface a defect outside its own scope (a
plan-vs-vision/drift gate, a review skill, or — most often — a `type:verify`
issue or an integration harness exercising the assembled system). This is
expected and desirable: it is the loop auditing itself. Handle it the **same
way every time**, regardless of which task or gate found it:

1. **Never inline-fix it.** Do not let the discovering task quietly absorb a
   fix outside its brief. The orchestrator never writes production code, and an
   unrelated task silently fixing behavior defeats the per-task gate rigor.
2. **File it as its own GitHub issue:**

   ```bash
   gh issue create \
     --title "<short-slug>: <one-line problem>" \
     --label "automation-<severity>,type:<...>,reviews:<...>" \
     --body "$(cat <<'EOF'
   Discovered-by: #<discovering-issue-number>, iter <N>, workflow #<W>
   ...
   <structured detail block — see type:verify section for template>
   EOF
   )"
   ```

   A fix touching auth/payments/security paths gets `reviews:security-review`.

3. **Severity decides the label** — production-fatal / blocking →
   `automation-crit` (jumps to top of the queue, above the discovering issue);
   high → `automation-high`; medium → `automation-medium`; low/cosmetic →
   `automation-low`. The severity-ordered query in Step 2a will pick the new
   issue first on the next invocation. Verify findings never go to
   `automation-verify` (that label is for verify tasks themselves).
4. **Wire the dependency only if it blocks _acceptance_.** If the new issue
   blocks the discovering issue's acceptance, edit the discovering issue body
   to add `- [ ] #<new>` to its task-list. The discovering issue stays open;
   on a future invocation Step 2b will skip it until the dep is closed. If
   the discovering task still has a self-contained deliverable, mark **that**
   part complete (close the discovering issue or comment "blocked portion
   spun out to #<new>") and preserve the blocked portion as an explicit,
   skipped spec (e.g. `describe.skip` with a comment naming the new issue)
   whose un-skipping is the new issue's acceptance — never delete or weaken
   it.
5. A genuine code-vs-vision contradiction still follows the `type:code`
   drift-review path below (a `type:decision` issue, not a `type:code` fix).

This makes the "prepend the blocker, resume after" mechanic an
orchestrator-wide invariant — not a behavior special to one issue.

### Kickoff comment + 👀 reaction (every type-flow, first action)

After Step 2d picks + optionally relabels the issue, **before any
per-type work begins**, the orchestrator posts a kickoff comment + a
`👀` reaction on the user comment it's acting on. This creates the
conversation/decision trail and the "processed" marker:

```bash
# 1. Identify the user comment being acted on (from Step 2d). Capture its id.
COMMENT_ID=<id>

# 2. Add the 👀 reaction — the "processed" marker for future runs
gh api -X POST /repos/<owner>/<repo>/issues/comments/$COMMENT_ID/reactions \
  -F content=eyes

# 3. Post the kickoff comment, citing the user comment by timestamp + plan
gh issue comment <N> --body "**Acting on @<user>'s <timestamp> comment.**

$([ \"<relabeled>\" = \"yes\" ] && echo \"Reclassified \\\`type:<old>\\\` → \\\`type:<new>\\\` based on comment intent.\")

Plan:
- <3-6 bullet plan for what's next>

I'll comment again when the work lands.

---
🤖 Conversation trail iteration: <short context note>."
```

If the issue body alone (no user comments) was the basis for pickup —
e.g. the very first iteration of an issue with no comments yet — the
kickoff cites the body instead: `**Acting on the issue body.**` No
reaction needed (there's no comment to mark).

The kickoff is **non-skippable** for any `type:*` flow. It's the
visible "I'm taking this" signal + the start of the conversation log.

### `type:code`

1. **Create the per-issue branch.** Every `type:code` task gets its own
   short-lived branch off `base_branch` (from config, default `main`):

   ```bash
   git fetch origin
   git switch -c automation/issue-<N>-<short-slug> origin/<base_branch>
   ```

   `<short-slug>` is derived from the issue title — kebab-case, ≤40 chars,
   alphanumeric + hyphens only. Capture the SHA you branched from (`git
rev-parse origin/<base_branch>`) for the implementer brief.

2. **Plan.** Single concern → draft the step list yourself. 2–4 areas → spawn
   a `Plan` subagent for it. (Track-sized was already split by effort scaling.)
3. **Plan-vs-vision gate.** Spawn a fresh-context review subagent with the
   _vision-review brief_ (`reference/task-brief.md` §"Vision-review brief"),
   baseline `vision_docs`: does this plan deliver what the vision intends
   _without contradicting it_? Verify any external-version claim (Node, library
   version, browser API) cites an authoritative source; unverified version
   claims fail the gate. If misaligned → revise the plan and re-gate. If it
   exposes a genuine vision ambiguity → file a new `type:decision` issue,
   write the artifact, flip this issue's label to `manual-decision`, stop.
   Catching divergence here is far cheaper than after code.
4. Fill `reference/task-brief.md` (implementer brief) with this issue's
   objective (from its body), acceptance, file scope, boundaries,
   `gate_commands` from the config, and — critically — the **per-issue
   branch name** from step 1 plus its **base commit SHA** so the implementer
   can rebase off the right base. The brief's "Stale-base worktree
   mitigation" section tells the implementer to `git fetch origin && git
rebase origin/<branch>` before starting and to report the observed HEAD.
5. Spawn the implementer via the **Agent tool**, `isolation: "worktree"`,
   `subagent_type: general-purpose`. The filled brief is the entire prompt.
6. On return, **do not trust the summary**. Inspect the actual diff in the
   returned worktree path. Run `gate_commands` in order against it.
7. Run each review skill in the issue's `reviews:*` labels; address findings.
8. **Drift-review gate.** Spawn a fresh-context review subagent with the
   vision-review brief: does the _actual merged behavior/diff_ match
   `vision_docs`? Classify the verdict:
   - **aligned** → proceed.
   - **doc-only drift** (behavior is fine; docs describe it wrongly/stalely)
     → proceed; reconcile the docs in Step 4b.
   - **behavioral contradiction** (code does the opposite of the vision, e.g.
     a gate enforced in the wrong place) → **do not open the PR.** File a
     new `type:decision` issue capturing the conflict. Evidence is the
     offending `file:line`, the vision excerpt it violates, and the precise
     question. Add `- [ ] #<new-decision>` to this issue's task-list (it stays
     open, blocked). Commit on the per-issue branch, push, comment on the
     issue with the blocker, stop. Never silently fix behavior or logic
     unattended.
9. If the task reveals real ambiguity or balloons at any point: stop, file a
   new `type:decision` (or `type:feel`) issue, flip this issue's label to
   `manual-decision` (or `manual-task`), comment the precise question, pause.
10. Commit on the per-issue branch, including any touched component READMEs
    and `vision_docs` reconciliations, recording review evidence per
    `review_evidence` (e.g. a `Reviewed-by:` trailer). The commit message
    body MUST include `Fixes #N` so GitHub closes the issue when the PR
    auto-merges.

### `type:decision`

Write/iterate `docs/decisions/<id>.md` on `branch`: framed options, tradeoffs,
a recommendation, explicit open questions. Commit. Then:

```bash
gh issue edit <N> --add-label manual-decision --remove-label automation-<sev>
gh issue comment <N> --body "Options doc: docs/decisions/<id>.md
Question: <one precise sentence the user needs to answer>"
```

Stop with a crisp summary of exactly what decision is needed (no need for the
user to scroll). On a future invocation, when the user has answered (a new
comment containing `Selected: <option>` or equivalent), flip the labels back
(`gh issue edit <N> --add-label automation-<sev> --remove-label manual-decision`),
execute the choice, and proceed.

### `type:feel`

Get the feature runnable (or have a subagent do so). Then:

```bash
gh issue comment <N> --body "How to try it: <exact commands or URL>
Question: <the ONE product question you want answered>"
gh issue edit <N> --add-label manual-task --remove-label automation-<sev>
```

Commit. Stop.

### `type:verify`

A task whose job is to **re-exercise the built system and self-expand the
issue queue** with whatever it finds (live e2e re-audits, integration sweeps,
doc-vs-reality reconciliations). It is the loop's auditing-itself mechanism
and the primary driver of the **verify → find → file issues → fix → re-verify**
cycle that converges the system. Designed so a first run can surface dozens of
problems without losing or churning any of them.

- **Always ranked last** (Step 2a `automation-verify` bucket): runs only when
  no other automation-\* bucket has an eligible issue, so it audits a
  _finished_ system and can never sit above the fixes it spawns. Multiple
  `automation-verify` issues order among themselves by `createdAt`.

- **Bring-up gate (workflow #0).** Before exercising any application
  workflow, prove the system is runnable: bring up dependencies (DB,
  fixtures, seed), start the server, hit a liveness probe. **A bring-up
  failure IS a finding** — file it as a discrete issue per the structured
  template below and stop; do not pretend later workflows "passed" on a
  dead stack. The verify issue's checklist must list bring-up as step 0.

- **Run every independently-runnable workflow per iteration; halt only when
  no further independent workflow is reachable.** "Sweep-blocking" ≠
  "workflow failing." If workflow N fails but N+1 is independent of N's
  outputs (e.g. N is parent billing, N+1 is the developer write path), run
  N+1 anyway and file findings for BOTH. Only stop the iteration when every
  remaining workflow's preconditions are unreachable due to an upstream
  failure (transitive: e.g. /sessions broken → /questions/ /break-sessions/
  /answers all unreachable). This maximizes findings per iteration and
  avoids many round-trips of "fix one trivial thing, re-run to discover the
  next one."

- **File EVERY finding from the iteration atomically.** Each iteration adds
  ONE new comment on the verify issue (the iteration checkpoint, with the
  sentinel block below) AND files each finding as its own GH issue in a
  single batch (`gh issue create ...` per finding). The verify issue's body
  task-list gets `- [ ] #<finding>` lines appended for every blocking
  finding (`gh issue edit <N> --body <updated>`). A killed orchestrator
  either committed/filed the whole batch or none — no half-state. Findings
  always carry `Discovered-by: #<verify-issue>, iter <N>, workflow #<W>` in
  their body.

- **Structured finding-issue body (REQUIRED — every spawned issue).** A
  verify-spawned issue is not actionable unless the implementer can fix it
  without re-investigating. Each new issue body must include:

  ```
  Discovered-by:   #<verify-issue>, iter <N>, workflow #<W> step <S>
  Severity:        blocking | high | medium | low
  Symptom key:     <method> <route> → <status> <code-or-fingerprint>
  Request:         <method/path/headers/body summary, exactly as sent>
  Observed:        <response status + body excerpt + relevant log lines>
  Expected:        <documented behavior; cite vision_docs / handler file:line>
  Suspected site:  <file:line(s) likely responsible — read the FULL
                    enclosing function/createRoute/block; do NOT cite a
                    grep-N-lines tail that may have truncated; include a
                    `grep -c <pattern>` sanity-count if the symptom is
                    about something being "absent" so the next reader can
                    distinguish "actually absent" from "grep window too
                    small">
  Repro:           <one-line repro command/curl that the verifier ran>
  ```

  This is the contract between the verifier and the fix-implementer — the
  fix issue must be self-contained.

  **False-positive prevention rule:** before filing a finding whose claim is
  _"X is missing"_, READ THE FULL TARGET BLOCK (not a truncated `grep -A N`
  slice that may cut off the evidence). A `grep -c <pattern>` on the whole
  file is a cheap absence-check. A false-positive caught later by Step 1's
  "verify against repo reality before acting" is closed as verify-only with
  a corroborating comment.

- **Severity → label mapping** (drives queue priority automatically, no
  per-task judgment):

  - `blocking` → `automation-crit` (next pickup). Added to this verify
    issue's task-list `- [ ] #N`.
  - `high` → `automation-high`. Not added to task-list unless it
    transitively blocks a downstream workflow.
  - `medium` → `automation-medium`.
  - `low` / cosmetic / doc-only → `automation-low`.

  `type:verify` issues never spawn other `automation-verify` findings (that
  label is reserved for verify-task issues themselves).

- **Duplicate detection — corroborate, don't duplicate.** Before filing,
  search for an existing open issue whose body contains a matching
  `Symptom key:` line (`gh issue list --search "Symptom key: <key>"`). If a
  match exists, **append a one-line corroboration comment** to it
  (`gh issue comment N --body "Re-observed <date>, iter <N>, workflow
#<W>"`) instead of filing a duplicate. Re-observations are signal that a
  prior fix didn't fully land — they should NOT be silently dropped.

- **Convergence sentinel — runaway guard.** Each iteration checkpoint
  comment on the verify issue records:

  ```
  iter N | workflows-passed P/T | findings-filed C |
  findings-fixed-and-verified F | blocked-on <#issue-or-none>
  ```

  If `findings-filed` grows for **3 consecutive iterations** without
  `findings-fixed-and-verified` also growing (no fixes landing, or fixes
  not actually fixing), the verifier **stops and files a `type:decision`
  issue** capturing the non-convergence and pauses for the user — better
  than silent churn. Likewise if `workflows-passed` ever **regresses**
  across iterations (a prior fix broke a prior pass), file a
  `type:decision` immediately (behavioral contradiction surfaced by the
  verifier).

- **Resumable prepend-gate.** On the next invocation after a blocking-fix
  closes, this verify issue becomes eligible again (its task-list dep
  satisfied) and **resumes from the latest checkpoint comment** — does NOT
  re-run workflows already marked ✅ in the prior iteration's comment.
  Already-✅ workflows are re-run only when the final pass is needed for
  close.

- **Closed — final pass.** Close the verify issue only after a final
  iteration where **every** workflow in the checklist is executed (no
  skipping past ✅) in one pass and all assertions hold, with zero new
  blocking findings (some non-blocking follow-ups may remain open and
  that's acceptable). Until then the issue stays open and the loop keeps
  prepend → fix → resume.

- **Never writes production code.** Even a one-line fix is a separate
  issue. Behavioral-contradiction findings still take the
  `type:decision` path.

### `type:external`

Comment on the issue describing the blocker (what the human/external party
must do); flip to `manual-task`:

```bash
gh issue comment <N> --body "Blocked on: <external action needed>"
gh issue edit <N> --add-label manual-task --remove-label automation-<sev>
```

Commit (no code), stop.

## Step 4 — Gates

Run `gate_commands` from `.claude/orchestrator-config.yml` in order; treat any
failure as root-cause work, not a bypass. Then run each required `reviews:*`
skill listed on the issue's labels. Record review evidence exactly as
`review_evidence` specifies (typically a `Reviewed-by:` git trailer on the
top commit, enforced by the project's pre-push hook and CI workflow). Never
bypass hooks or skip the configured gates. Honor any `CLAUDE.md` in the
project (commit hygiene, README maintenance).

## Step 4b — Reconcile vision & progress docs

Before recording on the issue, bring documentation in line with what actually
shipped — the same discipline as a doc-vs-reality audit, run every cycle so
docs never silently rot:

- Apply any **doc-only drift** the drift-review gate found.
- Update `vision_docs` to reality: status badges, roadmap "done/next" items,
  "what's done" lists, structure diagrams, dead links — only for what this
  task changed; do not rewrite the vision.
- If reality contradicts the vision and that is _intended_, the vision doc is
  what changes (with a one-line note why). If unintended, that was the
  behavioral-contradiction path in `type:code` step 7 — not a doc edit.

Doc edits here are yours to make directly (docs, not production code).

## Step 5 — Record & stop

For `type:code` issues that completed cleanly, follow the **per-issue PR +
auto-merge** flow (per `merge` config block):

1. **Push the per-issue branch:**

   ```bash
   git push -u origin automation/issue-<N>-<short-slug>
   ```

2. **Open a PR against `base_branch`** with `Fixes #N` in the body:

   ```bash
   gh pr create \
     --base <base_branch> \
     --head automation/issue-<N>-<short-slug> \
     --title "<issue title>" \
     --body "$(cat <<'EOF'
   Fixes #N

   ## Summary
   <one-paragraph summary of the diff>

   ## Gates
   - format ✅
   - build ✅
   - type-check ✅
   - test ✅ (<count> passed | <skipped> skipped)
   - lint ✅

   ## Reviews
   <skills run> — addressed inline; drift verdict ALIGNED (high).

   ## Notes
   <any uncertainty / boundary checks / honest limitations>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```

3. **Enable auto-merge** via the config'd strategy:

   ```bash
   gh pr merge <PR#> --auto --<merge.strategy> $(if merge.delete_branch then --delete-branch fi)
   ```

   The PR auto-merges when GitHub's required checks pass; `Fixes #N` auto-
   closes the issue; the per-issue branch auto-deletes (per `delete_branch`).

4. **Comment on the issue** with the PR URL + summary:

   ```bash
   gh issue comment <N> --body "PR #<PR#> opened — auto-merge enabled (<strategy>). $(date -u +%Y-%m-%dT%H:%MZ)"
   ```

5. **Do NOT close the issue manually.** `Fixes #N` + auto-merge handles
   closure; manual close is a duplicate signal.

6. **Do NOT use `--admin`.** If branch protection's required checks
   genuinely fail (test regression, real lint error), the PR sits open —
   that's the design. If checks are blocked by CI infrastructure (e.g.
   GitHub Actions billing), `gh pr view <PR#>` will show the BLOCKED
   state; comment on the issue explaining + escalate to the user. Do not
   bypass.

For non-code issue types, no PR is opened. Transition the issue per the
type-flow result:

- `type:decision` paused → already flipped to `manual-decision` (Step 3);
  comment with link to the options doc.
- `type:feel` paused → already flipped to `manual-task` (Step 3); comment
  with the "How to try it" block.
- `type:external` blocked → already flipped to `manual-task` (Step 3);
  comment with the blocker.
- `type:verify` mid-iteration → leave open; iter checkpoint comment is
  already appended. Only close on the final clean pass.
- Behavioral contradiction → leave open; the new `type:decision` issue is
  filed and linked.

### Closure-comment footer (every type-flow)

Every closure comment (`type:code` PR-opened, `type:decision` paused,
`type:feel` paused, etc.) ends with the conversation-trail footer that
cites the action chain so the issue thread reads as a back-and-forth:

```
---
**Conversation trail:**
- <user-comment-ts> — @<user> [<link>](...): "<one-line quote>" (👀-reacted)
- <kickoff-comment-ts> — orchestrator [kickoff](...): <one-line>
- <PR-or-doc-link-ts> — <PR #N / decision doc / iter checkpoint>
- <this-comment-ts> — **this comment**: <one-line outcome>
```

This makes the issue thread a self-contained decision trail — anyone
reading the issue 6 months later sees exactly what was directed, what
was acted on, and what shipped.

### Stop & schedule next (auto-chain)

At the end of each iteration, the orchestrator calls `ScheduleWakeup`
per the loop skill's dynamic-mode protocol — typically a 1200–1800s
delay with the same `/loop /vision-orchestrator` prompt. The next
firing re-enters the skill and runs the next iteration. This is the
operational default; `/loop /vision-orchestrator` is designed to be
fire-and-forget.

Context-bloat is real (conversation history accumulates across
`ScheduleWakeup` firings), but the **stateless-dispatch invariant**
(Hard Rules) guarantees correctness regardless: every iteration
re-derives all state from disk + GH, so Claude Code's automatic
context compaction — which is lossy by design — loses no load-bearing
state. Tokens cost more on later iterations; behavior doesn't change.

For genuinely unattended runs at scale (where in-session compaction
overhead is the bottleneck), the long-term answer is GH Actions cron +
a bot identity (Migration Stage 6, deferred per user). That bypasses
in-session `ScheduleWakeup` entirely.

End-of-iteration response template:

```
## Next iteration

PR #<N> opened; auto-merge enabled. Issue #<M> <closed | flipped to
manual-decision | …>.

ScheduleWakeup queued (1500s) to run the next /loop /vision-orchestrator
iteration. Next pickup will be <issue # or "queue empty — manual-* only">
(re-derived from disk + GH on next invocation; manual /clear is not
required — Claude Code auto-compacts).
```

End the turn with: what moved (PR number + issue number + outcome),
the drift verdict, the conversation-trail footer on the issue comment,
the `ScheduleWakeup` call, and (if anything is `manual-*`) the exact
ask.
