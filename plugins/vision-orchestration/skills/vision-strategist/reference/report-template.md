# Report template — the standing "roadmap command center" issue

The strategist rewrites the body of the `strategist-report` issue in place each
pass (living dashboard) and posts a short dated comment as the changelog. The
chat summary prints only the first two sections.

Render this structure:

---

```markdown
# 📊 Roadmap command center

_Last groomed: <UTC timestamp> by vision-strategist · Project: <name>_

## 🔴 ACTION REQUIRED (you)

The fewest things only **you** can do, ranked by how much downstream work each
unblocks. Everything else is either automated or waiting on these.

1. **#<N> — <title>** — <one line: what to do + what it unblocks (e.g. "register
   the @your-org npm org → unblocks #72 publish → unblocks 3 Track-J ports")>.
2. **#<N> — <title>** — <…>
3. …

> If this list is empty, the executor has a clear runway — nothing is gated on you.

## 📈 Health

- **Vision:** <X/Y milestones complete (NN%)> — nearest release: <milestone>.
- **Throughput:** <n issues closed in last <lookback> days>.
- **Meander index:** <0.NN> (<meta+verify+self-generated> of <total> recent
  closes were not vision-advancing). <"⚠️ above threshold — grooming/auditing
  faster than shipping" | "healthy">.
- **Goal-output-shipped:** <n>/<total closed> recent closes produced their
  declared deliverable on disk + validation evidence (when the project
  declares a "THE GOAL" anchor). <"⚠️ 0 deliverables shipped this window —
  loop is shipping internal-only code; install the goal-output toolchain or
  escalate" | "healthy">. Omit this line entirely when no goal anchor is
  declared.
- **Critical path head:** <#N — title> (the next thing that must ship).
- **Executor runway:** <n eligible-now issues> ready to pick.

## 🎯 Goal-output coverage

_Render this section only when the project's `vision_docs` declares a "THE
GOAL" anchor. Omit otherwise._

Per-deliverable status against the goal anchor — the at-a-glance answer to
"how far from THE GOAL are we?":

| Deliverable                  | Produced? | Authority cited? | Attack-fixtures pass? | Issue   |
| ---------------------------- | --------- | ---------------- | --------------------- | ------- |
| <e.g. "3-day pay-or-quit">   | ✅        | ⚠️ verify        | ❌ (no MtQ fixture)   | #…      |
| <e.g. "UD-100 complaint">    | ❌        | ❌               | ❌                    | #…      |
| <e.g. "UD-110 judgment">     | ⚠️ code   | ❌               | ❌                    | #…      |
| …                            |           |                  |                       |         |

Status legend: ✅ shipped + verified; ⚠️ partial (e.g., code lands but
deliverable not produced on disk, or only some fixtures pass); ❌ missing.
Rows come from the goal anchor's deliverable enumeration; ordering follows
the user-facing pipeline (notice → complaint → service → judgment → writ →
lockout in a PDF-shaped goal).

## 🧭 Critical path to <release milestone>

Ordered chain of unfinished work; each line is blocked by the one above it.

1. <#N — title> — <eligible-now | blocked-on #M | blocked-on-human>
2. …

## 🗂️ Backlog by status

| Status                                  | Count | Issues |
| --------------------------------------- | ----- | ------ |
| Eligible now (executor will work these) | n     | #…     |
| Blocked on another issue                | n     | #…     |
| Blocked on you (manual-\*)              | n     | #…     |
| Deletion proposed (your triage)         | n     | #…     |
| Off-track / no milestone                | n     | #…     |

## 🔧 What the strategist changed this pass

- **Reprioritized (n):** #N `automation-low → automation-high` (on critical
  path); …
- **Split (n):** #N → #c1, #c2 (was >400 LOC / mixed concerns); …
- **Combined (n):** #b folded into #a (meta-follow-up → parent DoD); …
- **Deletion proposed (n):** #N (off-scope / duplicate of #M / stale); …
- **None** in any empty category.

## 🧱 Risks & watch-items

- <e.g. "Track C entirely manual-\* → executor cannot make Phase-1 progress
  without human/Unity-license work">
- <e.g. "verify chain spawning follow-ups faster than they close">
```

---

Guidance for filling it:

- **ACTION REQUIRED is the headline.** It answers the user's recurring question
  ("what do I need to do?"). Rank by downstream-unblock count, not by age. Keep
  it to the genuinely human-gated few — if the list is long, the project is
  human-bottlenecked and that itself is the top finding.
- **Be specific in the changelog** — every relabel/split/combine/soft-delete
  with the one-line reason, so the issue thread is an auditable trail of why the
  backlog has its current shape.
- **Meander index** ties directly to the user's original concern (the orchestrator
  looping). Call it out explicitly when it's high, and name which off-path
  verify/meta issues you demoted because of it.
- **Goal-output-shipped** is the more expensive signal. When it's 0 while
  issues are still closing — the loop is shipping pure-internal code and
  producing no user-facing deliverable, the failure mode the four-axis
  vision check cannot detect. When this fires, the report's top
  recommendation should override every other recommendation with:
  "**Install the goal-output toolchain** (e.g. tectonic / pdf-filler MCP /
  the build pipeline) OR file a `type:decision` for the toolchain gap OR
  escalate to the user." Name the specific shipped issues whose
  deliverables were never produced as evidence.
- **Goal-output coverage table** is the at-a-glance scoreboard. Each pass,
  diff against the prior pass and call out which rows moved.
