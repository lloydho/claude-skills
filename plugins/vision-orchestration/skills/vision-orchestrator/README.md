# vision-orchestrator (importable Claude Code skill)

Progressively complete a large product vision with an **orchestrator +
scoped-subagent** model instead of one giant session. The task store is
**GitHub Issues**: one issue per task, labels carry priority + type + required
reviews, comments carry the per-task audit trail. The orchestrator queries the
queue via `gh` CLI, picks the top-eligible `automation-*` issue, advances it
**one task per invocation**, delegates the work to a worktree-isolated
subagent, and records the outcome on the issue. Every task is checked against
the vision twice — the **plan** before building and the **result** after
review — by independent subagents, so code that passes tests but drifts from
intent is caught, and vision docs are reconciled to reality each cycle. Humans
are pulled in only for ambiguous vision decisions (a design doc + issue
flipped to `manual-decision`), "how does it feel" signal (a runnable preview;
issue flipped to `manual-task`), or a code-vs-vision contradiction the loop
refuses to fix unattended — never for routine review.

This skill is **project-agnostic**. The only per-project surface is
`.claude/orchestrator-config.yml`.

## Contents

| File                                | Purpose                                                                                                    |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `SKILL.md`                          | The orchestrator (the `/vision-orchestrator` command).                                                     |
| `reference/orchestration-system.md` | The pattern's full rationale + architecture.                                                               |
| `reference/task-brief.md`           | Self-contained brief templates: implementer + vision-review (with stale-base + version-claim mitigations). |
| `reference/init.md`                 | One-time bootstrap procedure for a new project (config + labels + bootstrap issue).                        |
| `config.example.yml`                | Documented schema for `.claude/orchestrator-config.yml`.                                                   |

## Import into another project

1. Copy this whole directory to `.claude/skills/vision-orchestrator/` in the
   target repo:

   ```bash
   mkdir -p <target>/.claude/skills
   cp -R .claude/skills/vision-orchestrator <target>/.claude/skills/
   ```

   (Or vendor it via git subtree/submodule if you want updates.)

2. In the target repo, run:

   ```
   /vision-orchestrator init
   ```

   It detects the roadmap docs, the real gate commands, and any review-
   evidence convention, confirms autonomy with you, then writes
   `.claude/orchestrator-config.yml`, ensures the label set on the repo via
   `gh label create`, and files a `vision-bootstrap` issue (labeled
   `manual-decision`) as the starting landmark. Nothing is implemented during
   init.

3. Advance the vision one task at a time:

   ```
   /vision-orchestrator
   ```

   Re-invoke to continue. For hands-off cadence: `/loop <interval>
/vision-orchestrator`, or trigger headless `claude -p
"/vision-orchestrator"` from an external scheduler. (A held `/loop`
   session has idle cost — prefer external triggers for long unattended
   runs.)

## Optional: deterministic enforcement

The orchestrator enforces gates by judgment. To make them non-bypassable, add
`settings.json` hooks (`PreToolUse`/`Stop`) that shell-check tests/lint or a
review trailer. Hooks for _enforcement_; the skill for _logic_.

## Design

See `reference/orchestration-system.md`. Grounding:
[Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents),
[multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system),
[Claude Code subagents](https://code.claude.com/docs/en/sub-agents) /
[skills](https://code.claude.com/docs/en/skills) /
[hooks](https://code.claude.com/docs/en/hooks-guide),
[`gh` CLI](https://cli.github.com/manual/).
