# init — bootstrap the orchestrator in a new project

Run when `/vision-orchestrator init` is invoked or no
`.claude/orchestrator-config.yml` exists. This is a **one-time setup**. Do not
implement any vision task during init.

The task store is **GitHub Issues**; init does NOT create a markdown ledger.

## Steps

1. **Find the vision.** Ask the user (or detect) which docs are the
   source-of-truth roadmap (e.g. `docs/*plan*.md`, ADRs, a `ROADMAP.md`,
   PRD). List candidates and confirm with the user before recording them in
   the config.

2. **Detect the gate sequence.** Inspect the repo for the real verification
   commands (e.g. `package.json` scripts, `Makefile`, CI workflow). Propose an
   ordered `gate_commands` list (format/build/type-check/test/lint or the
   project's equivalent) and confirm with the user.

3. **Detect review-evidence convention.** Check for a pre-push hook / CI
   review gate / commit-trailer convention (e.g. `CLAUDE.md`, `.husky/`,
   `.github/workflows/`). Record it as `review_evidence`; if none, default to
   "run review skills, no trailer required".

4. **Confirm autonomy.** Ask the user: `checkpoint` (commit locally, stop,
   report — default), `auto-push`, or `auto-pr`.

5. **Write `.claude/orchestrator-config.yml`** if missing, using
   `.claude/skills/vision-orchestrator/config.example.yml` as a template.
   Fill in the answers from steps 1–4. The file is committed; it is the
   single per-project config surface.

6. **Ensure the label set exists on the repo.** Run each of the following.
   `gh label create` is idempotent if you supply `--force` (or use
   `gh label list` first to skip existing labels); a label that already
   exists with the right color is a no-op. Do not error out on existing
   labels — the bootstrap must be runnable on a repo that already has the
   labels (e.g. when re-running init).

   ```bash
   # Severity (orchestrator pickup; processed in this fixed order):
   gh label create automation-crit    --color b60205 --description "Production-fatal — orchestrator picks first" --force
   gh label create automation-high    --color d93f0b --description "High priority — orchestrator"                --force
   gh label create automation-medium  --color fbca04 --description "Normal priority — orchestrator"              --force
   gh label create automation-low     --color c2e0c6 --description "Cleanup / nice-to-have — orchestrator"       --force
   gh label create automation-verify  --color 5319e7 --description "Verify-task (always-last) — orchestrator"    --force

   # Manual (orchestrator skips):
   gh label create manual-decision    --color d876e3 --description "Human picks an option — orchestrator skips"               --force
   gh label create manual-task        --color bfdadc --description "Human acts (feel / external) — orchestrator skips"        --force

   # Type (mutually exclusive per issue):
   gh label create type:code          --color 1d76db --description "Routine implementation; full autonomous loop"             --force
   gh label create type:decision      --color 5319e7 --description "Vision-ambiguous; orchestrator writes options doc + pauses" --force
   gh label create type:feel          --color fef2c0 --description "Needs human product signal"                               --force
   gh label create type:external      --color e4e669 --description "Blocked on action outside the repo"                       --force
   gh label create type:verify        --color 0e8a16 --description "Re-exercises built system; self-expands the queue"        --force

   # Review skills required on a per-issue basis (any subset):
   gh label create reviews:review            --color 1d76db --description "review skill required before merge"          --force
   gh label create reviews:security-review   --color b60205 --description "security-review skill required before merge" --force
   gh label create reviews:simplify          --color 0e8a16 --description "simplify skill required after the change"    --force
   ```

7. **File the vision-bootstrap issue.** Create one `type:decision` issue with
   `manual-decision` (so the orchestrator skips it on the next invocation,
   leaving it for the user):

   ```bash
   gh issue create \
     --title "vision-bootstrap: outline the project's vision + first tasks" \
     --label "type:decision,manual-decision" \
     --body "$(cat <<'EOF'
   The vision-orchestrator has been bootstrapped on this repo. Before it can
   pick up real work, the user needs to:

   1. Confirm the vision docs in .claude/orchestrator-config.yml (`vision_docs`)
      are the authoritative roadmap.
   2. File the first batch of automation-* issues for the orchestrator to pick
      up. See `.claude/README.md` for the labeling quickstart.

   Suggested first tasks (file as separate issues, labels indicated):
   - <suggestion 1> — automation-high, type:code, reviews:review
   - <suggestion 2> — automation-medium, type:code, reviews:review
   - ...

   Respond with `Selected: <option>` or by filing the issues directly. Once
   one or more `automation-*` issues exist, run `/vision-orchestrator` to
   process them.
   EOF
   )"
   ```

   The bootstrap issue is `manual-decision`, so the orchestrator never picks
   it up itself — it sits as a permanent landmark / starting point for the
   user.

8. **Stop.** Report:
   - Config written to `.claude/orchestrator-config.yml`.
   - Labels ensured (list those created vs already-present).
   - Vision-bootstrap issue number.
   - Tell the user to file `automation-*` issues for their first tasks and
     then run `/vision-orchestrator` to advance one task, or
     `/loop <interval> /vision-orchestrator` for hands-off cadence.

## What the new config looks like

```yaml
# .claude/orchestrator-config.yml (created by init)

branch: <feature-or-main branch the orchestrator commits to>
vision_docs:
  - <glob/to/roadmap docs>
gate_commands:
  - <command 1>
  - <command 2>
review_evidence: <how to record review skills run; or "none">
autonomy: checkpoint # checkpoint | auto-push | auto-pr

# Storage substrate is always GitHub Issues since the gh-CLI migration.
storage: github-issues

# Severity ordering for pickup (orchestrator queries in this order until one
# returns an eligible issue):
labels:
  severity_order:
    - automation-crit
    - automation-high
    - automation-medium
    - automation-low
    - automation-verify
  manual_skip:
    - manual-decision
    - manual-task
```

See `.claude/skills/vision-orchestrator/config.example.yml` for a documented
template.
