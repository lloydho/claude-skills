# lloyd-skills — personal Claude Code marketplace

A personal [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
hosting one plugin, **`vision-orchestration`**, so the orchestration skills are
reusable across all my projects — including in **Claude Code on the web** (cloud
sessions clone the repo fresh and load plugins declared in a project's
`.claude/settings.json`; personal `~/.claude/skills/` is NOT available there).

## What's inside

`plugins/vision-orchestration/` bundles two skills:

- **vision-strategist** — the high-level orchestrator. Holds the whole vision +
  whole backlog and grooms it: reprioritize, split giant tasks, combine small
  ones, soft-remove dead ones; writes a standing report. Does not write code.
- **vision-orchestrator** — the executor. Picks one top-eligible `automation-*`
  issue per invocation, implements it via a worktree-isolated subagent, opens a
  per-issue PR, records the outcome.

Both read the consuming project's `.claude/orchestrator-config.yml`.

## Layout

```
.claude-plugin/marketplace.json              # marketplace catalog (lists the plugin)
plugins/vision-orchestration/
├── .claude-plugin/plugin.json               # plugin manifest
└── skills/
    ├── vision-strategist/SKILL.md  (+ reference/, config.example.yml)
    └── vision-orchestrator/SKILL.md (+ reference/, config.example.yml)
```

## Use it in a project

Add to that project's `.claude/settings.json` (commit it so cloud sessions pick it up):

```json
{
  "extraKnownMarketplaces": {
    "lloyd-skills": {
      "source": { "source": "github", "repo": "lloydho/claude-skills" }
    }
  },
  "enabledPlugins": {
    "vision-orchestration@lloyd-skills": true
  }
}
```

Then the skills are available as:

- `/vision-orchestration:vision-strategist`
- `/vision-orchestration:vision-orchestrator`

Each project still needs its own `.claude/orchestrator-config.yml`
(run `/vision-orchestration:vision-orchestrator init` to bootstrap one).

## Updating

Edit the skills here and push. Projects pick up changes on their next session
(the plugin re-resolves from this repo). Bump `plugins/vision-orchestration/.claude-plugin/plugin.json`
`version` when you want an explicit release marker; omit/auto behavior tracks the
commit SHA.
