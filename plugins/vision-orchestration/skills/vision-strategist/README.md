# vision-strategist (importable Claude Code skill)

The **high-level orchestrator** — the supervisor that sits above
[`vision-orchestrator`](../vision-orchestrator/README.md).

`vision-orchestrator` advances a large vision **one task per invocation**. That
narrow view is what keeps each task small and reviewable — but it also makes the
executor meander: it ships small, meta, and self-generated verification work
without keeping the whole project moving toward done. `vision-strategist` is the
fix. It holds the **entire** vision and the **entire** backlog, and periodically
**grooms** the backlog so the executor's next myopic pick is always the right
one.

It performs four operations on the GitHub-Issues backlog the executor reads:

| Operation        | What it does                                                                                                                                   | How (autonomy)                                                                                                                          |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Reprioritize** | Recompute `automation-<severity>` so critical-path / human-unblocking work ranks first (WSJF + critical-path + Theory-of-Constraints).         | Auto-apply (relabel — reversible).                                                                                                      |
| **Split**        | Break "too big" issues (>~400 LOC, mixed concerns, >5 acceptance criteria, decision+code, multi-session-no-done) into vertical-slice children. | Auto-apply (create children + umbrella the parent).                                                                                     |
| **Combine**      | Merge duplicates / same-module fragments / meta-follow-ups into a canonical issue's Definition of Done.                                        | Auto-apply (fold + re-point deps).                                                                                                      |
| **Remove**       | Soft-delete off-scope / duplicate / obsolete / stale / low-value issues.                                                                       | Auto-apply as **soft-delete**: strip `automation-*`, add `deletion-proposed` + rationale comment. **Never closes** — the human triages. |

Then it writes a **standing report** (a pinned "roadmap command center" issue it
rewrites each pass) and prints a **chat summary** whose headline is _ACTION
REQUIRED (you)_ — the few human-gated items blocking the critical path.

It is a **planner/groomer, not an implementer**: it never writes production
code, never opens a code PR, never runs the gates. Its only outputs are backlog
shape + the report.

## Contents

| File                             | Purpose                                                                              |
| -------------------------------- | ------------------------------------------------------------------------------------ |
| `SKILL.md`                       | The strategist loop (the `/vision-strategist` command).                              |
| `reference/grooming-criteria.md` | Research-derived decision rules + thresholds for all four operations, with sources.  |
| `reference/report-template.md`   | The standing-report structure (what the command-center issue + chat summary render). |
| `config.example.yml`             | The optional `strategist:` block for `.claude/orchestrator-config.yml`.              |

## How it pairs with the executor

The executor obeys `automation-<severity>` order (then oldest first). The
strategist **sets** those severities. That single seam is the whole integration:
reprioritization is expressed as relabeling, which the executor follows on its
next pick. Soft-removal works the same way — stripping `automation-*` makes an
issue invisible to the executor without closing it.

The strategist always grooms on a **longer cadence** than the executor
iterates, so it reorganizes a backlog that has actually moved. The skill works
under any of these deployments (it's setup-agnostic — see SKILL.md "Concurrent
operation"):

1. **Two parallel shells (default).** Executor and strategist each in their own
   `/loop`, different intervals. Concurrency is handled by the in-flight lock.

   ```
   /loop 1800s /vision-orchestrator   # shell A — fast
   /loop 6h    /vision-strategist     # shell B — slow
   ```

2. **Interleaved single loop (no concurrency).** One shell alternates: run N
   executor iterations, then one strategist groom. Strictly serial → zero race
   risk, no in-flight lock needed, one session's idle cost.

3. **External cron / headless (cheapest unattended).** Executor on its loop;
   strategist fired by a scheduler (GitHub Actions cron or system cron running
   `claude -p "/vision-strategist"`) every few hours. No held idle session for
   the strategist; same parallelism (and same in-flight lock) as option 1.

4. **Manual / on-demand.** Just run `/vision-strategist` when you want a fresh
   groom + report. Zero idle cost, full control, backlog drifts between runs.

All four rely on the same invariant: the executor is the single writer of one
in-flight issue at a time; the strategist re-derives state each run, steers via
labels, and skips the in-flight issue.

## Import into another project

1. Copy this directory to `.claude/skills/vision-strategist/` in the target
   repo (it shares `.claude/orchestrator-config.yml` with `vision-orchestrator`).
2. Optionally add a `strategist:` block to the config (see
   `config.example.yml`); sane defaults apply if absent.
3. Run `/vision-strategist`. On first run it creates the `deletion-proposed` and
   `strategist-report` labels and the command-center issue if they don't exist.

This skill is **project-agnostic**: the only per-project surface is
`.claude/orchestrator-config.yml` + the `vision_docs` it points at.

## Design

The four operations and their criteria are grounded in: WSJF (Reinertsen/SAFe),
Critical Path Method + topological scheduling, Theory of Constraints (Goldratt),
INVEST + vertical-slice story splitting, HTN / plan-and-execute decomposition,
the empirical small-PR↔bug-rate data, Cognition's single-threaded-writes
principle, and SWE-agent/BabyAGI loop-detection findings. Full attribution in
`reference/grooming-criteria.md`. Orchestration framing follows Anthropic's
[Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
(orchestrator-workers + evaluator-optimizer).
