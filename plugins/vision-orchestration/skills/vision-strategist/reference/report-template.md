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
   the @eduplay npm org → unblocks #72 publish → unblocks 3 Track-J ports")>.
2. **#<N> — <title>** — <…>
3. …

> If this list is empty, the executor has a clear runway — nothing is gated on you.

## 📈 Health

- **Vision:** <X/Y milestones complete (NN%)> — nearest release: <milestone>.
- **Throughput:** <n issues closed in last <lookback> days>.
- **Meander index:** <0.NN> (<meta+verify+self-generated> of <total> recent
  closes were not vision-advancing). <"⚠️ above threshold — grooming/auditing
  faster than shipping" | "healthy">.
- **Critical path head:** <#N — title> (the next thing that must ship).
- **Executor runway:** <n eligible-now issues> ready to pick.

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
