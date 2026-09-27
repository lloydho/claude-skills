---
name: writing-style
description: House writing style for any prose we ship — UI copy, labels and status chips, emails, error and empty states, marketing pages, READMEs, developer docs, commit messages. Use before writing or editing any user-facing or developer-facing text, and when reviewing a diff that changes such text.
---

# Writing style — don't sound like a robot

Rules for any prose we ship, in any project: app copy, marketing pages, emails, docs, commit messages.
The goal is writing that reads like a person wrote it, not a chatbot.

This isn't about gaming detectors. The AI defaults — hedged, padded, relentlessly even-toned — are
just worse writing. Cutting them makes the copy shorter and clearer, which is the actual win.

**Project overrides win.** If the repo has a `WRITING_STYLE.md`, `EDITING.md`, `docs/documentation-style.md`,
a `.claude/rules/*` copy rule, or a machine-enforced ban list (e.g. `copy-standards.ts`), read it and let it
override this file on any conflict. This file is the floor, not the ceiling.

## How AI writing gives itself away

- **Filler vocabulary.** A cluster of words shows up far more in machine text than human text: seamless,
  robust, comprehensive, leverage, utilize, delve, crucial, pivotal, underscore, testament, tapestry,
  landscape, realm, elevate, streamline, holistic, empower, foster, garner, showcase, myriad, plethora,
  intricate, meticulous, vibrant, ensure (as filler), facilitate, paramount, invaluable, unparalleled,
  ecosystem (as metaphor). None are banned outright, but each one earns its place or gets cut.
- **The "not just X, it's Y" flourish.** "It's not a class, it's a community." Negative parallelism used
  for drama. Almost always cuttable. Same for rhetorical "X, not Y" and "X, without Y" in headings,
  descriptions, and buttons — keep a contrast only when a real eligibility, policy, safety, or legal
  boundary would otherwise be unclear.
- **Comma-linked slogan fragments.** Two or more short fragments, clauses, or generic claims staged side
  by side for a payoff: "Every plan, side by side." "More choice, less hassle." "Built for families,
  designed for winter."
- **Rule of three.** Three adjectives or phrases in a row when one would do: "fast, reliable, and easy to
  use." Fine once; a tic when every sentence does it.
- **Empty connectives.** Moreover, Furthermore, Additionally, In conclusion, It's worth noting that,
  When it comes to. They announce a thought instead of having one.
- **Hollow -ing tails.** Sentences ending in a clause that restates the point: "...cutting wait times,
  highlighting our focus on speed." The tail adds no information.
- **Reflexive hedging.** Every claim wrapped in "may," "can help," "often," "generally" until nothing is
  actually said. Take a position.
- **Formula conclusions.** "Despite the challenges, the future looks bright." The wrap-up paragraph that
  summarizes what you just read.
- **Mechanical formatting.** Bolding every key term, Title Case On Every Heading, an em dash in every
  other sentence, `**Label:** description` on every bullet. Punctuation as decoration.
- **Even, featureless tone.** No contractions, no short sentences, no asides, no opinion. Uniform and
  frictionless, which reads as lifeless. If any two paragraphs could swap positions unnoticed, both are padding.

## What to do instead

- Every sentence must add information the reader needs. If it only repeats a heading, line item, amount,
  or button, remove it. Use that space to explain the consequence or next step.
- Write the way you'd say it out loud. Use contractions. Vary sentence length — a three-word sentence
  after a long one does real work.
- Be specific. "Refunded in 24 hours" beats "we handle refunds promptly." Concrete detail is the single
  strongest signal a human wrote it.
- Cut the qualifier unless the uncertainty is real and load-bearing. If it's real, name it precisely
  ("directional, vendor-sourced") rather than vaguely.
- Have a point of view. Recommend, don't survey. It's fine to say something is the better option and why.
- Bold and em dashes for meaning, not rhythm. If you bolded three things in a paragraph, two of them
  probably don't need it.
- Delete the throat-clearing. "In today's world," "When it comes to X" can almost always go.
- Sentence-case headings unless it's a proper noun.
- Keep button labels to a concise action phrase. Use plural category labels when a listing represents
  multiple variations of a class, program, event, or product. Write team biographies in the third person.
- Let the copy be a little uneven. A small aside or a blunt word is what people write and models don't.

## Context is not copy

The brief, ticket, conversation, schema, and implementation contain facts the writer needs. That does not
mean the reader needs those facts. Do not turn the model's context window into narration on the page.

Include a fact only when it helps the reader make a decision, complete the current action, avoid a mistake
or cost, or understand a necessary consequence that isn't already obvious. If removing the sentence changes
none of those, remove it.

> Before: `You're paying $280 online today. Invitations send only after payment succeeds.`
> After: remove it. The order summary already shows the amount, and the reader doesn't need the system's
> invitation timing.

Say the thing once. If a sentence restates the previous one in different words, delete one of them.
Transactional email is the strictest case: the reader wants the fact, the consequence, and the link.

## Don't stage fragments side by side

Do not join short fragments, clauses, or generic claims just to create a two-beat slogan. This includes
comma, period, colon, dash, and line-break versions — changing the punctuation preserves the same canned
rhythm:

- `Every plan, side by side.`
- `More choice. Less hassle.`
- `The mountain: reimagined.`
- `Built for families — designed for winter.`
- heading `Every plan` followed by subhead `Side by side`

Write one natural sentence with a finite verb. Name the actor, action, object, or reader consequence the
slogan is hiding. Don't "fix" the pattern by adding *and* or swapping punctuation.

> Before: `Every plan, side by side.` → After: `Compare every plan in one table.`
> Before: `Everything included, nothing hidden.` → After: `The price includes equipment rental and all listed classes.`

This doesn't ban commas or concise fragments. Ordinary lists, addresses, introductory or subordinate
phrases, appositives, necessary qualifiers, and short UI labels that name a real state are fine. The test
is whether the copy uses two balanced or detached units as a staged payoff instead of stating the fact directly.

## Labels that don't mean anything

The patterns above are about *sounding* like a robot. This one is worse: copy that reads fine but tells the
reader nothing. It happens most in short UI text — status chips, badges, empty states, column headers —
where the writer knows the internal state and forgets the reader doesn't.

The tell: **the label names an internal state or a piece of jargon, but not what it means for the person
reading it.** A real shipped example: `Afterschool — starts at Discovery`. It's the `pending` membership
state and means *this membership isn't active yet; it begins once the skier does their Discovery lesson.*
None of that survives. "at Discovery" reads like a place, "starts" has no clear subject, and nothing says
the membership is inactive.

How to catch it:

- **Read the label as someone who doesn't know the schema.** If understanding it depends on knowing what
  `status: "pending"` means, the reader doesn't have that — the label has to carry it.
- **Prepositions doing too much work.** "at Discovery," "on hold," "in review," "by term" — a bare
  preposition plus a noun the reader has to decode is usually a state leaking into the UI.
- **Can they act on it?** A status the reader can't act on or ask about is vibes. "Is it active? When does
  it change? What unblocks it?" If the label answers none of those, it's not done.
- **Jargon is fine once it's grounded.** Tie the term to an event and a tense the reader can place:
  "starts after Discovery lesson," not "at Discovery."
- **Say what's true, not the plausible version.** The fix above is *not* "billing starts after Discovery" —
  the first term is usually prepaid at signup, so billing already happened.

> Before: `Payment — pending` → After: `Charge sent — waiting on the bank (usually clears in a minute)`
> Before: `Coverage: by term` → After: `Covered through the paid term, then it stops`

Schema and view-model types are the source of truth for what a state means. Read the field comments before
naming a state in the UI.

## Money, numbers, and dates

- **Never print a ledger word to a customer** — `credit`, `comp`, `quota`, `waived by`. Say what makes the
  line cheap and whose benefit it is: `Free with Lloyd's membership`, not `Paid with credit`.
- **`Free` vs `$0`.** Say `Free` only when the customer never paid. A line that nets to zero because money
  already moved says `$0`. Before shipping a zero, find the row's paid-for-it case and check which it is.
- **Never claim irrevocability the contract doesn't have.** "Non-refundable," "all sales final," "no
  refunds" are legal claims, not copy choices — say "minimum commitment" or "committed term" instead
  unless a lawyer has signed off on the stronger wording for that jurisdiction.
- Prices are whole dollars (`$280`), not cents-padded (`$280.00`). Four figures carry a thousands
  separator (`$1,200`, not `$1200`). Symbol form (`$150`), never `150 dollars`.
- Dates in copy are long form (`July 28, 2026`) — never `7/28/2026`, never `Jul 28, 2026`.
- Em dash (`—`) for an aside, en dash (`–`) for a range, never a spaced hyphen.
- One surface must not contradict another. If two pages describe the same right, price, or deadline, they
  say the same thing, with the number read from one source rather than hand-typed twice.

## Developer-facing docs

Extra rules for **integration and reference documentation a third party reads to use our software**:
`docs/`, `README.md`, `MIGRATION.md`, SDK reference, developer-portal pages. The register is a competent
engineer telling another engineer what to do. The test for every sentence: does it say what to do, what
will happen, or what something is? Delete anything else.

These rules do **not** apply to internal guides that argue a position rather than instruct — style
guides, ADRs, design rationale, `CLAUDE.md` and `.claude/rules/*`, postmortems. Those keep the house
voice above, first-person plural and all. Judge them by the general rules, not this section.

- **Voice.** Imperative, second person, active. "Import the plugin." "Call `Initialize` once at startup."
  Never "the developer should" or "it is recommended that." Start every step with a verb. Use "is" and
  "has" for descriptions, not "serves as" or "acts as." State rationale in one sentence immediately before
  the rule it justifies — rationale never gets its own paragraph.
- **Structure.** Prerequisites in a bulleted list before the first step, no prose preamble. Number
  sequential steps; give parallel tasks their own headings. Name headings after the task, sentence case.
  Put each code block directly under the sentence introducing it. End integration guides with a
  verification section: what to run, what output means success.
- **Callouts.** Three labels, one meaning each. **Warning** — skipping this breaks the integration; name
  the exact scenario and consequence. **Note** — a constraint the reader must know but can't act on.
  **Tip** — optional, saves time. A specific Warning outranks any typography; don't emphasize by
  scattering bold, writing REQUIRED in table cells, or adding exclamation points.
- **Code samples.** Comments state intent, not mechanics. `ALL_CAPS_UNDERSCORE` for values the reader must
  replace. Self-contained and minimal, one concept per block.
- **Prohibited.** Exclamation points in prose. "simply," "just," "easily." Marketing adjectives in
  instructional text ("powerful," "seamless," "cutting-edge," "best-in-class"). Narrative about how or when
  a feature was built — docs are timeless; that belongs in commit messages and CHANGELOGs. First-person
  plural ("we designed this so that..."). Arrow chains as prose. Em-dash asides mid-instruction.
  Rhetorical questions and transition filler ("Now that you've done X..."). "Note that" as an inline
  softener — promote it to a Note callout or delete it. Emoji in headings (README status badges excepted).

## Quick self-check before shipping copy

1. Search for the filler words above. Each hit: cut or justify.
2. Any "not just X, it's Y," "X, not Y," or "X, without Y"? Rewrite as a plain statement.
3. Any comma-linked or line-broken slogan fragments? Replace the staged rhythm with one concrete sentence.
   Don't preserve it with different punctuation.
4. Three-in-a-row lists — keep one if it's doing work, trim the rest.
5. Could a reader act on this, or is it vibes? Add the concrete detail.
6. Every status label, chip, and empty state: read it as someone who doesn't know the schema. If it leaks
   an internal state or bare jargon without saying what it means for them, rewrite it.
7. Is a sentence there because the reader needs it, or because the writer knew it? Remove context narration.
8. Money lines: no ledger words, `Free` vs `$0` correct, no unbacked irrevocability claim, formatting right.
9. Does this contradict any other surface describing the same fact?
10. Read it aloud. If you'd never say it, rewrite it.

## Examples

> Before: "Our seamless onboarding empowers families to effortlessly unlock a world of skiing, ensuring a
> comprehensive and tailored experience."
> After: "Sign up once. Add everyone in your household, pick a plan for each, and you're booked — nothing
> to rebook or renew."

> Before: "It's not just a lesson — it's a journey toward becoming a confident, capable, and accomplished skier."
> After: "A coach gets you on the slope and helps you find your level before classes start."
