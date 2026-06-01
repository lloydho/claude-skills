# Grooming criteria — the decision rules for the four operations

These are the mechanical rules the strategist applies. They are derived from
established planning, agile-sizing, and autonomous-agent literature (sources at
the bottom). Apply them in this order every pass: **build the model →
reprioritize → split → combine → remove → report**. Splitting before combining
avoids merging fragments of a task you're about to break up anyway.

The single biggest lever (confirmed across the literature): **dependency-aware,
constraint-first ordering — find the critical path / current bottleneck and
drive work down it, keeping each change small and vertically sliced.** For a
single-agent executor this collapses to one rule the strategist enforces by
shaping the backlog: _always make the executor's next-eligible pick the
smallest item that unblocks the most downstream work (or unblocks the human)._

---

## The model (build this first, every pass)

1. **Map every open issue to a vision milestone/track.** Use `vision_docs` (the
   roadmap) as the milestone list. An issue that maps to **no** milestone is an
   off-track candidate (feeds Remove).
2. **Build the dependency DAG.** Edges come from `- [ ] #N` task-list lines in
   issue bodies. Topologically sort. The **critical path** is the longest chain
   of unfinished work from "now" to the nearest release milestone (e.g. "Phase
   1 done"). Compute each issue's **downstream-blocked count** = how many open
   issues transitively depend on it.
3. **Classify each open issue** into exactly one bucket:
   - `eligible-now` — has `automation-*`, no `manual-*`, all deps closed.
   - `blocked-on-issue` — has an open `automation-*`/`manual-*` dep.
   - `blocked-on-human` — carries `manual-*` (decision/task/external).
   - `off-track` — maps to no milestone, or is a self-generated follow-up with
     no traceable value.
4. **Compute health + meander signals** from recently-closed issues (default
   last 7 days):
   - `vision-advancing` = closed issues that map to a milestone and shipped
     user-visible behavior or a roadmap track.
   - `meta/verify/self-generated` = orchestrator self-modification, test-harness
     scaffolding, or "Discovered-by:" follow-ups.
   - **Meander index** = `meta+verify+self-generated / total closed`. A
     sustained index > ~0.6 means the loop is grooming/auditing itself faster
     than it ships vision — the canonical meandering signal.
   - **Goal-output-shipped** (new) = closed issues whose declared
     `PDF-deliverable:` (or analogous goal-anchor output field) actually
     produced the deliverable on disk + the validation evidence the goal
     anchor enumerates (cited authority + self-check + attack-fixture
     pass). When the project's vision-doc declares a "THE GOAL" anchor
     this metric is the canonical "are we moving toward the user
     outcome" signal. **Goal-output-shipped is the metric the
     four-axis vision check cannot infer from labels — it requires
     actually looking at whether the artifact exists.** A sustained
     `goal-output-shipped = 0` while issues are still closing IS the
     meander pattern even if `Meander index` is low, because every
     close is "internal code" not "user-facing deliverable."
5. **Check goal-link contract compliance** (when the project's vision-doc
   declares a "THE GOAL" anchor). Scan each open `type:code` issue body
   for the goal-link fields the anchor enumerates (typically `Goal-link:`,
   an output-deliverable field, `Validation:`, `Done-when:`). Issues
   missing those fields are candidates for Operation 4 (soft-remove) per
   the new "Goal-link-absent" criterion. This is the structural filter
   that prevents code-for-code's-sake issues from sitting in the
   eligible-now queue indefinitely.

---

## Operation 1 — Reprioritize

The executor picks strictly by `automation-<severity>` then oldest `createdAt`.
**The strategist's only prioritization lever is the severity label.** Rewrite
severities so the executor's myopic pick aligns with the critical path.

Compute, for each `eligible-now` / soon-eligible issue:

```
wsjf = (value + time_criticality + risk_or_unblock_value) / effort_estimate
```

(score each term 1–5 ordinally; WSJF — Reinertsen / SAFe.) Then assign severity:

| Assign severity     | When                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------- |
| `automation-crit`   | On the critical path AND `downstream-blocked count ≥ 3`, OR production-fatal.                                 |
| `automation-high`   | On the critical path, OR removes a **human** dependency (relieves the true constraint), OR top-quartile WSJF. |
| `automation-medium` | Ready, off critical path, mid WSJF.                                                                           |
| `automation-low`    | Bottom-quartile WSJF AND nothing depends on it.                                                               |
| `automation-verify` | Verify/audit tasks only (always last by design).                                                              |

**Hard overrides (apply after the table):**

- **Critical-path-first:** never let an off-critical-path issue outrank a ready
  critical-path issue. (Theory of Constraints: off-constraint work just "creates
  piles of half-finished work.")
- **Human-unblock-first:** an issue whose completion unblocks a `manual-*` item
  the human is waiting on outranks ordinary on-path work.
- **Confidence floor:** demote (or send to Split/Remove) any issue whose value
  can't be stated as an increment of user-visible progress.

Apply via relabel only — fully reversible, safe to auto-apply.

---

## Operation 2 — Combine small / related tasks

**Merge two (or more) issues into one canonical issue when ANY:**

- They share the same primary file/module **AND** neither delivers standalone
  testable value alone (INVEST "S"/"I"; SRP "things that change for the same
  reason belong together").
- One is strictly a sub-step of the other.
- Acceptance-criteria overlap > ~70% (near-duplicate).
- The candidate is a **meta/verification follow-up** ("add test for X",
  "document X", "verify X") of a still-open parent X → fold it into X's
  Definition of Done. **This directly counters meandering** — it stops
  self-generated micro-tasks from becoming their own queue.
- Two issues write to the **same files/region** (Cognition single-threaded-
  writes: parallelizing same-file writes "makes implicit decisions that
  conflict" — merge into one serial unit).

**Only merge while the bundle stays under the Split ceiling (Op 3).** Combine is
the inverse of Split; never let it recreate a giant task.

**How (safe to auto-apply):** pick the canonical issue (oldest, or the one with
dependents). Append the absorbed issues' acceptance criteria to its body.
Re-point any `- [ ] #absorbed` dep lines in other issues to the canonical issue.
Then soft-remove each absorbed issue via Operation 4 with reason
`merged into #<canonical>`.

**Pitfalls:** don't merge two items that each individually deliver value just
because they're topically related (hurts incremental delivery / small PRs).
Never merge across a dependency edge in a way that forms a cycle.

---

## Operation 3 — Break apart giant tasks

**Split an issue when ANY of these "too big" triggers fire:**

- Estimated diff > ~400 LOC (empirical review-quality cliff: review quality
  drops sharply > 200 LOC; > 400 LOC catches fewer bugs; < 400 LOC ≈ 40% fewer
  production defects — Google/Cisco/SmartBear data).
- Touches > ~3 modules/files, or mixes concerns (SRP violation).
- Has > ~5 acceptance criteria, or criteria spanning multiple architectural
  layers that could each ship independently.
- **Mixes a decision + an implementation** → split the decision/spike out as its
  own `type:decision` issue (the implementation children depend on it).
- Bundles happy-path + edge-cases + docs + tests as separable deliverables.
- **The executor has touched it across > 2 sessions without reaching "done"**
  (long-task-low-progress is the strongest looping signal — SWE-agent).

**How (HTN / plan-and-execute decomposition; safe to auto-apply):** decompose
the compound issue into **vertical slices** — each child touches the layers it
needs and leaves the build green and demonstrable. Create child issues
(`type:code`, appropriate severity, `reviews:*` inherited), add `- [ ] #child`
lines to the parent body, and convert the parent into a tracking umbrella
(strip its own `automation-*` so the executor works the children, not the
umbrella). Keep all writes to a single file inside ONE child (conflict
avoidance).

**Pitfalls:** **never split horizontally** (DB-only / API-only / UI-only shards
aren't demonstrable). Don't over-decompose — children "have no context of each
other's work," so each must be self-contained, and coordination overhead grows
with child count. Each child must independently satisfy INVEST.

---

## Operation 4 — Remove (soft-delete; never close)

**Per user policy, the strategist NEVER closes or deletes an issue.** To
"remove" an issue it:

1. Strips all `automation-*` labels (the executor stops picking it up).
2. Adds the `deletion-proposed` label.
3. Posts a comment: `Deletion proposed: <reason>. <evidence>. Reopen for
automation by re-adding an automation-* label, or close to confirm.`

The human triages the `deletion-proposed` queue and is the only actor that
actually closes. This makes removal fully reversible.

**Propose deletion when ANY:**

- **Duplicate** — acceptance overlap > ~70% with another open issue (link it).
- **Obsolete** — its preconditions no longer hold (referenced code/feature
  changed or was removed) or it's superseded by completed work.
- **Off-scope** — traces to no milestone in `vision_docs` (anti-scope-creep;
  many self-generated agent follow-ups fail this).
- **Low-value** — bottom WSJF/RICE AND no open dependents.
- **Stale** — untouched > T days (default 30) AND no open dependents AND not on
  the critical path.
- **Absorbed** — merged into a canonical issue by Operation 2.
- **Goal-link-absent** (new) — when the project's vision-doc declares a
  "THE GOAL" anchor and the issue is `type:code`, the issue body lacks
  the goal-link fields the anchor enumerates (commonly `Goal-link:`,
  output-deliverable, `Validation:`, `Done-when:`). These are the
  pure-internal "code-for-code's-sake" candidates that meander.
  Soft-remove with the rationale citing the missing fields + a link to
  the goal anchor. The human triages: either the fields get added (and
  the issue is reinstated by re-adding `automation-*`) or the issue is
  confirmed off-track (closed). `type:verify`, `type:decision`,
  `type:feel`, `type:external` are exempt — only `type:code` needs the
  fields.

**Pitfalls (hard guards):**

- **Check in-edges first.** NEVER propose deletion of an issue that has open
  dependents — and note that stripping `automation-*` makes dependents treat it
  as resolved, which would falsely unblock them. If an issue is a dependency of
  any open issue, do NOT soft-delete it; instead re-point or close the dependent
  relationship explicitly first.
- **Sunk-cost bias** — judge on _remaining_ value, not effort already spent.
- **Never** soft-delete a critical-path item.
- Verify "done" before treating something as superseded — agents "confidently
  state they had completed a task when they had not" (BabyAGI failure mode).

---

## Definition of Done & anti-meandering (cross-cutting)

- Encode "done" as **tests pass + every acceptance criterion met + (when the
  vision-doc declares a THE GOAL anchor) the declared deliverable observed +
  validation evidence on file**, and fold meta/verification follow-ups into
  their parent's DoD (Op 2) so they cannot spawn an endless self-generated
  micro-queue.
- If the **meander index** exceeds ~0.6 for two consecutive passes, the report's
  top recommendation is "pause net-new verify/meta work; drive the critical
  path," and the strategist demotes off-path verify/meta issues to
  `automation-low`.
- If **goal-output-shipped is 0** for two consecutive passes WHILE other
  issues are closing (i.e., the loop is shipping internal code but no
  user-facing deliverables), the report's top recommendation overrides
  the meander recommendation with: "**we're shipping internal-only code;
  no goal-anchor deliverable has been produced this groom window.** Install
  the goal-output toolchain (e.g. tectonic/pdf-filler for PDF goals; the
  build pipeline for service-deployment goals), OR file a
  `type:decision` issue capturing the toolchain gap, OR escalate to the
  user." This guard is what catches "the loop ships LaTeX templates that
  have never been compiled, AcroForm composers that have never been
  filled" — the failure mode the four-axis vision check cannot detect
  from labels alone.
- Keep every executor change small and vertically sliced (Op 3) — this is the
  second-biggest lever and the one that most directly buys "minimal bugs,"
  because small changes are easier to review and to revert.

---

## Sources

- WSJF — Reinertsen, _Principles of Product Development Flow_; SAFe WSJF
  (`framework.scaledagile.com/wsjf`), Black Swan Farming.
- Critical Path Method / topological scheduling — Kelley & Walker 1957;
  `en.wikipedia.org/wiki/Topological_sorting`.
- Theory of Constraints — Goldratt, _The Goal_ / _Critical Chain_; ToC Institute.
- INVEST + vertical-slice story splitting — Bill Wake; Humanizing Work splitting
  guide; Applied Frameworks vertical-slice.
- Single-Responsibility Principle — Robert C. Martin.
- HTN planning; plan-and-execute / LLMCompiler DAG planning — LangChain
  "Planning Agents".
- Small-PR / bug-rate data — Google/Cisco/SmartBear (review quality vs PR size).
- Single-threaded writes / multi-agent conflict — Cognition, "Multi-Agents:
  What's Actually Working".
- Loop detection / DoD in coding agents — SWE-agent, Devin; BabyAGI failure
  modes (IBM overview).
- Backlog hygiene / scope-creep pruning — Agile Alliance backlog refinement;
  RICE (ProductPlan); MoSCoW.
- Orchestrator-workers / evaluator-optimizer patterns — Anthropic, "Building
  Effective Agents".
