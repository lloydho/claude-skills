# Vision Orchestration System (spec)

A pattern for **progressively completing a large product vision** with Claude
Code, using an orchestrator + scoped subagents instead of one giant session.
Project-agnostic. This file ships inside the `vision-orchestrator` skill and
explains the _why_; `SKILL.md` is the executable _how_.

---

## 1. Problem

A single long session that "builds the whole thing" fails predictably: context
bloats, the model loses the end-to-end thread, early guidance gets compacted
away, and one bad turn poisons everything downstream. Vision/roadmap docs are
also _ambiguous_ and _change_ — some decisions can only be made after a human
test-drives the result and reports how it _feels_.

## 2. Core ideas

1. **State lives outside the conversation, in a queryable store.** Sessions are
   ephemeral. The cross-session memory is a set of **GitHub Issues** carrying
   labels (severity, type, reviews), bodies (current spec), and comments
   (chronological audit trail). Closed issues = done work; open
   `automation-*`-labeled issues = the queue. `gh` CLI is the access surface.
2. **The orchestrator holds the vision; subagents hold the work.** The
   orchestrator never implements. It decomposes, delegates one scoped task,
   verifies the result, advances the queue. (Anthropic's orchestrator-workers
   pattern: a lead agent decomposes and synthesizes; workers execute.)
3. **One task per invocation.** Progress is one issue picked, processed,
   transitioned (closed, flipped to `manual-*`, or left open mid-verify) — not
   a session running for hours. Re-invoke (or loop) to continue.
4. **Agents are the code-review gate; humans are the _product_ gate.** Automated
   review (multiple agents) catches code defects better than a human skim. Pull
   the human in only for (a) ambiguous vision decisions (`type:decision` →
   flipped to `manual-decision`) and (b) "how does it feel" signal
   (`type:feel` → `manual-task`) — never for routine code review.
5. **Trust but verify the issue and the subagents.** Both drift and over-claim.
   The orchestrator re-checks repo reality before acting and inspects the
   actual diff after.
6. **Vision drift is detected, not assumed away.** Gates that prove _code is
   correct_ (tests, lint, review) do not prove _code matches the vision_. So
   every task is checked against the vision twice — the plan before building,
   the result after — by independent fresh-context subagents. Drift is
   triaged: stale docs get reconciled automatically; code that contradicts the
   vision becomes a new `type:decision` issue, never an unattended "fix". This
   is the loop doing to itself what a doc-vs-reality audit does to a codebase.

## 3. Architecture

```
            ┌─────────────────────────────────────────────┐
            │  ORCHESTRATOR  (top-level session / skill)   │
            │  • reads config, queries gh, verifies vs repo│
            │  • picks next unblocked automation-* issue   │
            │  • classifies by type:* label                │
            │  • spawns ONE scoped implementer per issue   │
            │  • verifies diff, runs gates, comments+closes│
            └───────────────┬─────────────────────────────┘
                            │ Agent tool, isolation: worktree
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │implementer│  │ Plan     │  │ review   │   (subagents — NO nesting:
        │ subagent  │  │ subagent │  │ subagents│    a subagent cannot spawn
        └──────────┘  └──────────┘  └──────────┘    subagents)
              │
              ▼
        GitHub Issues (queue + audit trail) ◀── single source of cross-session state
```

### 3.1 The artifacts

| Artifact                                          | Role                                                                                                                                                                                   |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **GitHub Issues** (the task store)                | One issue per task. Labels carry state (`automation-<sev>`, `type:*`, `reviews:*`). Body = spec. Comments = audit trail. Closed = done.                                                |
| **`.claude/orchestrator-config.yml`**             | Per-project config: `base_branch`, `merge` (per-issue PR + auto-merge), `gate_commands`, `review_evidence`, `vision_docs`, `autonomy`, label schema (severity ordering + manual-skip). |
| **Orchestrator** (`SKILL.md`)                     | User-invocable skill that runs exactly one queue step.                                                                                                                                 |
| **Implementer brief** (`reference/task-brief.md`) | Self-contained template the orchestrator fills per subagent (objective, output format, scope, boundaries, branch/HEAD for stale-base mitigation).                                      |
| **Init** (`reference/init.md`)                    | One-time bootstrap: create config, ensure label set, file the vision-bootstrap issue.                                                                                                  |

### 3.2 Task taxonomy (drives orchestrator behavior)

Carried by mutually-exclusive `type:*` labels on the issue:

- **`type:code`** — routine. Fully autonomous: worktree → implementer subagent →
  required review subagents → orchestrator verifies diff → merge → comment
  outcome → close. No human.
- **`type:decision`** — ambiguous / architecturally significant. Orchestrator
  writes no production code. It produces/iterates a design doc
  (`docs/decisions/<id>.md`: options, tradeoffs, recommendation, open
  questions) on a branch, comments the doc link + the precise question on the
  issue, and flips the label `automation-<sev>` → `manual-decision`. The user
  replies via an issue comment (e.g. `Selected: Option C`); on the next
  invocation the orchestrator flips back to `automation-<sev>` and executes
  the choice (often splitting it into `type:code` child issues).
- **`type:feel`** — needs human product signal. Orchestrator gets the feature
  runnable (dev server / preview deploy / mock URL), comments a concise "how
  to try it" note + the one question being asked, and flips the label to
  `manual-task`. User responds on the issue.
- **`type:verify`** — re-exercises the _built_ system and self-expands the
  issue queue with what it finds. Each iteration adds a checkpoint comment on
  the verify issue; findings are filed as NEW issues with
  `Discovered-by: #<verify-issue>` in their bodies and the right
  `automation-<severity>` label. Load-bearing properties (see SKILL.md
  §`type:verify` for the full protocol): **always ranked last** (the
  `automation-verify` bucket is the last severity-ordered query) so it audits
  a finished system and can never sit above the fix issues it spawns;
  **bring-up is workflow #0** (a bring-up failure is itself a finding);
  **runs every independently-runnable workflow per iteration** (a workflow
  failing doesn't halt the sweep unless every remaining workflow's
  preconditions are upstream-unreachable — maximizes findings per round
  trip); **files every finding atomically** (no half-state if interrupted)
  with a **structured body block** (Symptom key, Request, Observed, Expected,
  Suspected site, Repro) so the fix-implementer doesn't have to
  re-investigate; severity → label mapping is automatic (blocking →
  `automation-crit` + added to verify issue's task-list; high/medium/low →
  `automation-<sev>`); **duplicate detection** corroborates by Symptom key
  rather than filing duplicates; a **convergence sentinel** stops and files a
  `type:decision` if findings keep filing without fixes landing (or if a
  workflow regresses across iterations) — prevents silent runaway. This
  collectively turns the verifier into the engine of the
  **verify → find → file issues → fix → re-verify** convergence cycle.
- **`type:external`** — blocked on a human action outside the repo. Orchestrator
  flips to `manual-task` with a comment describing the blocker; never fakes
  it.

**Severity ordering** (replaces the old "tier 1 / tier 2 / tier 3" mechanism):
the queue is processed by `automation-<sev>` prefix in this fixed order —
`automation-crit > automation-high > automation-medium > automation-low >
automation-verify`. Within a severity bucket, `createdAt` ascending is the
tie-break. The orchestrator's Step 2a issues five `gh issue list --label
automation-<sev>` queries in this order and takes the first eligible candidate.

Two semantics fall out of the severity ordering for free:

- A `type:verify` issue is **always ranked last** because `automation-verify`
  is the last bucket queried — it can never sit above the fixes it spawns,
  and so a verifier cannot be blocked by its own discoveries.
- A `type:verify` issue that surfaces a `blocking` finding files an
  `automation-crit` issue, which immediately preempts every other queue
  position by virtue of being in the first severity bucket. The verifier's
  CRIT discovery preempts routine work without any special table-jumping
  logic.

**Discovered work is an orchestrator-wide invariant.** Any task, implementer,
or gate that surfaces an out-of-scope defect → the orchestrator files it as its
own correctly-labeled GH issue and (if it blocks acceptance) adds a
`- [ ] #<new>` task-list line to the discovering issue, **never** inline-fixes,
and preserves any blocked acceptance as an explicit skipped spec. The
"prepend the blocker, resume after" mechanic is this invariant applied to a
`type:verify` issue — it is not special to one task; an ordinary `type:code`
issue or harness that trips over a production bug is handled identically.

A `type:code` issue can also be **kicked back to a new `type:decision` issue
by the drift-review gate**: if the shipped behavior contradicts the vision,
the orchestrator does not close-and-move-on — it files a new `type:decision`
issue with the evidence and the precise question, links it from the original
issue's task-list (which stays open), and stops. The next invocation picks the
next unblocked issue, so an overnight run keeps making progress on other
fronts while the contradiction waits for a human.

The orchestrator must **self-reclassify mid-flight**: if a `type:code` task
turns out ambiguous or oversized, it stops, files a `type:decision`/`type:feel`
issue (or splits into child `type:code` issues), flips the current issue's
label to `manual-*`, and pauses — instead of guessing.

## 4. Execution loop (one invocation)

1. Read `.claude/orchestrator-config.yml`. **Verify against repo reality** —
   `base_branch`, clean tree, fetch — never trust labels blindly.
2. Pick the highest-severity eligible open issue (severity-ordered `gh issue
list` queries; skip any with `manual-*` labels or with open
   `automation-*`/`manual-*` task-list deps).
3. **Create the per-issue branch** `automation/issue-<N>-<short-slug>` off
   `base_branch`. All work for this iteration lands on this branch only;
   never on `base_branch` directly.
4. **Effort scaling**: trivial → do it directly, no subagent; single-concern →
   1 implementer; 2–4 areas → Plan subagent then implementer; track-sized →
   split into child issues (task-list lines) and stop.
5. Act by `type:*` (see §3.2). For `type:code`: plan → **plan-vs-vision gate**
   (fresh-context subagent) → implementer subagent → inspect the **actual
   diff** (not the summary) → gate sequence → required review skills →
   **drift-review gate** (fresh-context subagent vs `vision_docs`) → commit.
6. **Reconcile docs to reality**: apply doc-only drift, refresh the vision
   docs for what shipped. Docs never silently rot.
7. **Push the per-issue branch, open a PR against `base_branch`** with
   `Fixes #N` in the body, then **enable auto-merge** per `merge.strategy`
   (squash by default). Comment on the issue with the PR URL. The PR
   merges when GitHub's required checks pass; `Fixes #N` auto-closes the
   issue; the branch auto-deletes per `merge.delete_branch`. Do NOT use
   `--admin`. If branch protection blocks the merge, the orchestrator
   escalates rather than bypasses.
8. **Stop.** One task per invocation keeps every context small and disposable;
   the new gates live _inside_ that single task, not as extra invocations.

## 5. Human-feedback protocol

The only interruptions:

- **`type:decision` (flipped to `manual-decision`)**: framed options + a
  recommendation in `docs/decisions/<id>.md`, linked from an issue comment.
  User answers in an issue comment; orchestrator turns the resolution into
  `type:code` children.
- **`type:feel` (flipped to `manual-task`)**: exact commands/URL + the single
  question being asked, in an issue comment. Feedback becomes a follow-up.

Code correctness is **never** a human-feedback reason — that is the automated
review subagents' job.

## 5.1. Conversation-driven dispatch

**The user interface is "flip the label + comment."** The orchestrator
treats the issue body as canonical context and the **comment thread as
the brief**. Body edits are NOT how the user requests work; they're how
the spec was originally captured.

On every pickup, the orchestrator scans the issue's comments for the
**latest unprocessed user comment** — one that has no `👀` reaction
from the orchestrator's identity and no orchestrator response citing
it. That comment's intent governs the dispatch. If the inferred
`type:*` doesn't match the current label, the orchestrator
**auto-relabels** before any work and notes the swap in the kickoff
comment.

The "processed" marker is the `👀` reaction. The decision trail is the
kickoff comment (which cites the user comment by timestamp) +
subsequent orchestrator comments + closure footer. The issue thread
reads as a back-and-forth: user direction → orchestrator kickoff →
orchestrator deliverable → user pick → orchestrator follow-up. Anyone
opening the issue 6 months later sees the entire decision chain
without reading the codebase.

Two consequences this design accepts:

1. **The body never grows.** The user doesn't have to keep re-editing
   it to refine the spec; refinements are comments. This also means
   the orchestrator never edits issue bodies it didn't author.
2. **Comments are addressable units of work.** A single issue can host
   multiple rounds of "research → pick → implement → review →
   iterate" — each round a comment chain — without needing to spawn a
   new issue per refinement.

The `type:*` label is mutable per-pickup; it reflects the LATEST
classification, not the originally-filed one. An issue can start as
`type:decision` (research), get user-picked, flip to `type:code`
(implement the pick), flip to `manual-decision` (pause for next
question), and so on across its lifetime — the label is a state
indicator, not a permanent classification.

## 5.2. Session management & context cost (stateless dispatch)

The orchestrator's correctness is **stateless across invocations** —
every dispatch decision is re-derived from disk + GH on every run. This
is validated by spawning a fresh-context subagent (no prior
conversation), giving it only the documented inputs (skill docs,
config, vision-docs, gh issues, git state, CLAUDE.md), and confirming
it reaches the same Step 1 + Step 2 + Step 3-plan conclusion the
in-session orchestrator does. Validated 2026-05-20 PM (#53).

**Operational chaining via `ScheduleWakeup` is the default.** Each
iteration ends with a `ScheduleWakeup` call (1200–1800s typical delay,
per the loop skill's dynamic-mode protocol) queuing the next
`/loop /vision-orchestrator` invocation. The orchestrator is
fire-and-forget — no manual user intervention between iterations.

Context-bloat is a real cost concern: conversation history accumulates
across `ScheduleWakeup` firings within a session. Token-cache pressure
starts after ~5 minutes of staleness; later iterations pay more for
the same dispatch decision. The cost-bound on it is **Claude Code's
automatic context compaction** — which is lossy by design, but the
stateless-dispatch invariant makes that loss safe:

- Every iteration re-reads `.claude/orchestrator-config.yml`, runs the
  live `gh issue list` + `gh issue view --comments`, checks git state.
- A compacted/summarized prior turn loses no load-bearing state, only
  redundant context that would have been re-derived anyway.
- The Hard rule "Stateless across invocations" + the Step 2d mandatory
  live-fetch directive structurally guarantee this: in-context memory
  of comments is NEVER trusted; the live `gh` query is the source of
  truth.

We considered a "clear-and-continue" protocol (orchestrator stops
without `ScheduleWakeup`; user runs `/clear && /loop` between
iterations) and briefly shipped it in PR #55 — but the user reversed
that direction immediately: _"Clearing manually is not ok. Let's give
up compacting context and rely on claude's auto compact for now."_
(2026-05-21T00:31:31Z on #53; the same iteration's failure to fetch
that comment before kickoff motivated the Step 2d mandatory-fetch
fix.) The clear-and-continue protocol was reverted in PR for #56
(this section). The unattended-loop UX is too important to gate on
manual `/clear`.

For genuinely-unattended large-scale runs, the long-term answer is
Migration Stage 6 (GH Actions cron + bot identity) — that bypasses
in-session `ScheduleWakeup` entirely.

If you find yourself wanting "the orchestrator should remember X from
my last iteration": that's a sign X should be on the issue (comment,
label, task-list item, decision doc), not in conversation memory.

## 6. Stale-base worktree mitigation

Harness worktrees from `Agent` with `isolation: "worktree"` may branch off
the harness default (typically `main`) rather than the orchestrator's working
branch — the implementer's diff against the orchestrator's branch then looks
catastrophic (every commit the branch added since `main` shows as a
deletion). The mitigation is two-sided:

- **Implementer side** (in `reference/task-brief.md` `⚠️ Stale-base worktree
mitigation (read first)` section, at the top of the brief): on first
  action, the implementer runs `git fetch origin && git rebase
origin/<branch>` and reports the observed HEAD. If the rebase produces
  conflicts it cannot resolve cleanly, the implementer STOPS and reports
  rather than guessing.

- **Orchestrator side**: when filling the implementer brief, include the
  explicit `branch` name AND the latest commit SHA on origin/<branch>. The
  implementer's first task is to verify it lands on that SHA before doing
  any work. If the implementer reports a different HEAD after rebase, the
  orchestrator investigates before trusting the result.

This is cheap (one fetch + one rebase per implementer cycle), works around
the harness default, and keeps the diff-inspection step honest.

## 7. External-version-claim verification

Review gates check _internal_ consistency (plan matches vision; diff matches
vision) but can miss _external_ grounding — claims about library/runtime
versions, browser APIs, or RFC semantics that nobody validated against an
authoritative source. The mitigation lives in two places:

- **Plan-vs-vision gate** (vision-review brief): for any external-dependency
  version claim (Node, library version, browser API, RFC), the plan must cite
  the authoritative source (CHANGELOG entry, MDN page, RFC). Unverified
  version claims fail the gate; the orchestrator will not approve a plan that
  asserts a version-floor without a citation.

- **Implementer brief** (UNCERTAINTY field): if the implementer cannot verify
  a version claim themselves, they mark the claim UNCERTAIN in their report.
  The orchestrator treats unverified version claims as failed and re-spawns
  with a corrected plan rather than merging the unknown.

This is cheap, surfaces a real class of error caught by the review gates, and
shifts the cost from "find out after deploy" to "find out before merge."

## 8. Claude Code mechanics this relies on (grounded)

- **Subagents (Agent tool).** Subagents **cannot spawn subagents** — the
  orchestrator must be the top-level session. Fresh context per subagent; the
  only input channel is the brief string, so the brief must be self-contained.
- **Worktree isolation.** `isolation: "worktree"` runs the subagent on an
  isolated git worktree; empty result auto-cleans, otherwise the path + branch
  return for the orchestrator to inspect and merge. A broken subagent cannot
  corrupt the main tree or sibling tasks. Mitigate stale-base branching (§6).
- **Skill as entrypoint.** `.claude/skills/<name>/SKILL.md` with YAML
  frontmatter; invoked as `/<name>`. Loads at session start; reads the config +
  queries gh.
- **`gh` CLI.** The task store is accessed via `gh issue list / view / edit /
comment / create / close`. Authentication is the user's local `gh auth login`
  (or a GitHub App token in unattended runs); no separate API client lives
  in-tree.
- **Cadence.** Manual re-invocation; or the `/loop` skill (`/loop <interval>
/vision-orchestrator`); or unattended via headless `claude -p` on an
  external trigger. A held `/loop` session has idle cost — prefer external
  triggers for long autonomy.
- **Hooks.** `settings.json` hooks (`PreToolUse`/`PostToolUse`/`Stop`/…)
  enforce shell-checkable invariants (tests, lint, review trailer) independent
  of model judgment. Hooks for _enforcement_, the orchestrator for _logic_.
- **Memory.** No built-in cross-session store; the GitHub issue queue is the
  pattern. Durable rules belong in `CLAUDE.md` (survives compaction).

## 9. Failure modes & mitigations

| Failure                                 | Mitigation                                                                                                                               |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Issue labels drift from reality         | Re-verify repo state every run before trusting `open`/`closed` + labels.                                                                 |
| Subagent over-claims success            | Inspect the real diff + re-run gates; never accept the summary as proof.                                                                 |
| Orchestrator context bloat              | Delegate research/impl to subagents; keep only config + vision summary + the current issue + its deps; one task per run.                 |
| Whole track handed to one agent         | Effort-scaling rule: track-sized work splits into child issues (task-list lines) first.                                                  |
| Ambiguity bulldozed into code           | `type:decision`/`type:feel` labels + mid-flight reclassification (flip to `manual-*`) force a human checkpoint.                          |
| Nested-subagent attempt                 | Orchestrator stays top-level; briefs never tell a subagent to spawn agents.                                                              |
| Idle loop cost                          | Prefer external trigger over a held `/loop` session for unattended runs.                                                                 |
| Code passes gates but misses the vision | Two independent drift gates (plan + result) vs `vision_docs`; correctness gates can't see intent.                                        |
| Vision docs rot as code moves           | Step-5 doc reconciliation every cycle; doc-only drift is auto-applied, not deferred.                                                     |
| Stale-base worktree branching           | Implementer brief's `git fetch && rebase origin/<branch>` at start; orchestrator-side branch+SHA in brief; verify post-rebase HEAD (§6). |
| External-version claims unverified      | Plan-vs-vision gate requires CHANGELOG/MDN/RFC citation for any version-floor claim; implementer reports UNCERTAIN if unverifiable (§7). |
| Unattended "fix" of a contradiction     | Behavioral contradiction → new `type:decision` issue + stop; the loop never edits behavior/logic to resolve drift on its own.            |

## References

- Anthropic — Building Effective AI Agents:
  https://www.anthropic.com/research/building-effective-agents
- Anthropic — How we built our multi-agent research system:
  https://www.anthropic.com/engineering/multi-agent-research-system
- Claude Code — Subagents: https://code.claude.com/docs/en/sub-agents
- Claude Code — Skills: https://code.claude.com/docs/en/skills
- Claude Code — Hooks: https://code.claude.com/docs/en/hooks-guide
- Claude Code on the web: https://code.claude.com/docs/en/claude-code-on-the-web
- GitHub CLI (`gh`): https://cli.github.com/manual/
