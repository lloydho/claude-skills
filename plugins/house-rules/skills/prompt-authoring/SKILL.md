---
name: prompt-authoring
description: How to write and edit prompts that a Claude model will run — system prompts, per-task templates, judge and gate prompts, classifiers, structured-output prompts. Use before writing a new prompt, editing an existing prompt template, or debugging a prompt that misbehaves (over-triggers, over-explains, ignores a rule, or drifts on edge cases).
---

# Prompt authoring

Read this before you write or edit a prompt. Consult it *first* rather than iterating blind — most
prompt bugs are one of the known failure modes below, and the fix is structural, not another adjective.

Model IDs, pricing, parameter names, and API mechanics are **not** here — load the `claude-api` skill
for those, and never answer from memory.

## Know which generation you're targeting

Claude 5-generation models (Fable 5, Opus 5, Sonnet 5) need substantially less scaffolding than 4.x did.
Anthropic removed over 80% of Claude Code's own system prompt for these models with no measurable loss
on coding evals. That changes what good looks like:

- **Delete repeated instructions.** Earlier models sometimes needed the same rule stated twice, in two
  layers. Now the repeat is dead weight — and worse, two near-identical rules that drift apart become a
  contradiction the model resolves arbitrarily.
- **Delete overconstrained rules.** Rigid mechanical bans ("never write multi-line comments") cost more
  than they buy; these models handle contextual judgment well. Keep hard rules for genuine foot-guns and
  real safety lines.
- **Prefer expressive interfaces over instructions.** If you're writing a tool, encode the constraint in
  the parameter definition and a well-named field rather than a paragraph telling the model what not to
  pass. Tool descriptions and system prompts should not both carry the same guidance.
- **Progressive disclosure over one big prompt.** Split long conditional guidance into skills or
  reference files that load when relevant, instead of a prompt that pays for every branch every call.

Older models still in a pipeline (Haiku 4.5, anything 4.x) do want the more explicit treatment below.

## Structure

- State the role once, in the system prompt. A per-task template adds only its task — don't restate the
  persona in every file.
- One concern per tag: `<role>`, `<task>`, `<rules>`, `<output>`, `<examples>`. Use consistent,
  descriptive names, and nest only where the content is genuinely hierarchical.
- Match the prompt's own style to the output you want. If you want plain text or raw JSON back, write
  the prompt as plain text — no markdown, no emoji. The model mirrors what it sees.

## Instructions

- **Be explicit about scope.** Models follow instructions literally and won't silently generalize a rule
  to a sibling case. Say "every field" or "each bullet"; don't imply it.
- **Say what to do, not what not to do.** "Reply in one warm sentence, then one question" beats "don't
  write paragraphs." Reserve negative rules for real foot-guns.
- **Give the why in a clause** — "…so a network blip never traps the user." The model generalizes
  correctly from the reason to cases you didn't enumerate. This is the highest-value sentence you can add.
- **Don't shout.** "CRITICAL / you MUST / NEVER EVER" makes models over-trigger and drag the behavior
  into unrelated turns. Normal phrasing ("Use…", "Prefer…") steers fine. Keep hard "never" for the few
  real safety lines.
- **Two rules that contradict each other are worse than one vague rule.** Before adding a rule, grep the
  prompt for the one it might conflict with.

## Examples — the highest-leverage lever

- Include 3–5, each in `<example>`, all inside `<examples>`.
- Make them **relevant** (mirror real inputs) and **diverse** (cover the borderline and edge cases, not
  three flavors of the easy one).
- Prefer showing the good output over describing the bad.

## Judges, gates, and classifiers

- **Define the bar concretely, in terms of the downstream goal.** Never with vague adjectives — "specific
  enough," "be generous," "be strict." The model's idea of those won't match yours. A good bar reads like
  a question you could answer about a single input.
- **Strictness is context-dependent.** Encode that in per-case examples, not a global switch.
- **Decide which way it fails, and say so.** Every gate has an asymmetric cost. Name the cheaper error
  and instruct the model to prefer it when unsure — and make the parser fail the same direction, so a
  stray token can't hard-block the user.
- For a judge prompt the examples *are* the calibration. Put the borderline cases in the set with the
  verdict you want, so the model copies your line instead of inventing its own.

## Structured output

- Show the exact schema in an `<output>` block; current models match it reliably.
- Prefer a tool/structured-output call over free-text JSON when the API offers one — validation happens
  at the tool layer and the model retries on mismatch, instead of you parsing a near-miss.
- Keep the parser fail-open unless a wrong value is dangerous.

## Length

State it explicitly — "2–3 sentences, under 40 words," "one short nudge." Models size responses to
perceived complexity and will over-explain otherwise.

## Before you ship a prompt change

Re-validate against real model calls, not your reading of the diff. If the project has an eval harness
or golden set, run it; a prompt edit that looks obviously safe is exactly the kind that regresses an
edge case you didn't enumerate.
