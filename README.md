# lloyd-skills — shared Claude Code marketplace

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
that carries the rules and skills my projects share, so every collaborator's agent
gets the same version. This includes **Claude Code on the web**: cloud sessions clone
the repo fresh and load plugins declared in a project's `.claude/settings.json`, but
they can't see anyone's personal `~/.claude/`.

The repo is public, so collaborators need no GitHub access to install from it.

## Plugins

### `house-rules`

The rules every project follows, whichever org owns it.

- **`house-rules:writing-style`**: the house style for any prose we ship, from UI
  copy to commit messages. A repo's own `WRITING_STYLE.md`, `EDITING.md`, or copy
  ban list overrides it.
- **`house-rules:prompt-authoring`**: how to write and edit prompts a Claude model
  runs.
- **`code-reviewer`** (Opus) and **`writing-reviewer`** (Sonnet): read-only review
  agents for code and prose.

Project-specific rules still belong in each repo's `CLAUDE.md`. Plugins carry
skills, agents, and hooks, but not `CLAUDE.md` text.

### `vision-orchestration`

Two-tier autonomous project orchestration over a GitHub-Issues backlog.

- **`vision-strategist`** holds the whole vision and backlog and grooms it:
  reprioritize, split giant tasks, combine small ones, soft-remove dead ones. It
  writes a standing report and doesn't write code.
- **`vision-orchestrator`** picks one top-eligible `automation-*` issue per
  invocation, implements it through a worktree-isolated subagent, opens a per-issue
  PR, and records the outcome.

Both read the project's `.claude/orchestrator-config.yml`. Run
`/vision-orchestration:vision-orchestrator init` to create one.

## Layout

```
.claude-plugin/marketplace.json        # catalog listing both plugins
plugins/house-rules/
├── .claude-plugin/plugin.json
├── skills/writing-style/SKILL.md
├── skills/prompt-authoring/SKILL.md
└── agents/code-reviewer.md, agents/writing-reviewer.md
plugins/vision-orchestration/
├── .claude-plugin/plugin.json
└── skills/vision-strategist/, skills/vision-orchestrator/
```

## Use it in a project

Add this to the project's `.claude/settings.json` and commit it. Anyone who opens
the repo and trusts the folder is prompted to install the plugins.

```json
{
  "extraKnownMarketplaces": {
    "lloyd-skills": {
      "source": { "source": "github", "repo": "lloydho/claude-skills" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "house-rules@lloyd-skills": true
  }
}
```

Add `"vision-orchestration@lloyd-skills": true` only in projects that use the
orchestration skills.

## Updating

Edit the files here and push. `house-rules` sets no `version`, so installs track
the latest commit and pick up changes on their next marketplace update.
`vision-orchestration` pins a `version`, so bump it in its `plugin.json` to release
a change.
