# Writing Standard for FastSvelte Docs

Public documentation for FastSvelte, built with MkDocs Material, deployed at
https://docs.fastsvelte.dev. Product behavior is defined by the dev monorepo at
`~/workspace/fastsaas-workspace/fastsaas/` (separate clone; relative links to it
do not resolve from here).

## Audience

A buyer who has cloned the kit and wants to ship. Assume competence with Python,
FastAPI, and Svelte. Do not assume familiarity with *our* choices (Sqitch, the DI
container, the webhook-driven billing model). Explain our decisions, not the
frameworks.

## Page shape

The template to copy is `docs/features/billing.md` (shape, not punctuation; it
predates the em dash rule below):

1. Open with 2-4 sentences saying what this is and how it works. No "In this
   section, we will..." throat-clearing.
2. Then the steps or the behavior.
3. Then detail, with rationale pushed into collapsible blocks (below).

Two genres, two voices:

- **Guides** (`docs/guides/`) are tutorials. Imperative step headers ("Create the
  migration", "Run it"), a checkpoint after every stage, a short **Recap** bullet
  list at the end.
- **Features and reference** (`docs/features/`, `docs/reference/`) are
  explanations. Define each term in plain words before using it. State trade-offs
  and give a recommendation instead of listing options neutrally.

## Voice

Modeled on fastapi.tiangolo.com, minus the emoji:

- Second person, active: "you create", "you will see". Never "the user should".
- Before every terminal output, screenshot, or JSON response, set the
  expectation: "You will see:". After every runnable stage, say what success
  looks like so the reader can verify before continuing.
- Introduce a concept as a problem before the solution ("Right now the client
  can pick the `id`. We shouldn't allow that.") rather than announcing the
  feature.
- Grow one example across a page instead of switching examples; when a snippet
  changes, show it again and say what changed.
- Paragraphs of 1-3 sentences. Bold the one word a skimmer must catch, not whole
  phrases.

## Does not read as AI-generated

Enforceable tells; check drafts against this list:

- No em dashes in published copy. Periods, commas, parentheses.
- No emoji.
- No filler transitions: "Additionally", "Moreover", "Furthermore", "Overall".
- No "simply", "easily", "seamlessly", "powerful", "robust".
- No closing summary paragraph that restates the section (a guide's Recap bullet
  list is the only sanctioned recap).
- Concrete over generic: name the file, the command, the error message.

## Where detail goes

The test: remove the sentence and reread. If the page still feels complete, the
sentence is not load-bearing and moves out of the reader's path:

- **Short skippable detail (1-3 sentences): a visible admonition box.** The box
  itself tells the reader "good to know, not required", so they can skip it
  without wondering if they missed a step. Use the types consistently:
    - `!!! tip` for a better or faster way to do what the steps already do.
    - `!!! note` for a clarification or a naming alias ("a plan is what Stripe
      calls a product").
    - `!!! info` for background on how something works under the hood.
    - `!!! warning` is the exception: a pitfall that costs the reader if
      missed. Warnings must be read, so keep them rare or they stop working.
- **Longer rationale, edge cases, and "why we chose this": a `??? note "..."`
  collapsible block** (`pymdownx.details`, already enabled in `mkdocs.yml`).

Steps and behavior always stay inline. A reader following the steps should
never scroll past a paragraph they don't need.

## Verify before documenting

Every behavior claim gets checked against `fastsaas/` before it is written, and
the doc cites the real file. Never write from memory: the docs once gave three
different DI wirings (tutorial: Singleton repo + Factory service;
architecture reference: Factory repo + Singleton service) while the real
`fastsaas/backend/app/config/container.py` uses Singleton for both. That is what
skipping this rule produces.

## FastSvelte / FastReact parity

Backend content is identical to `fastreact-docs` modulo framework names.
Frontend content diverges only where the framework genuinely forces it. The
parity table lives in `~/workspace/fastsaas-workspace/fastsaas/CLAUDE.md`; link
to it, don't restate it.

## Workflow

Docs are written in short pieces. Draft a section, stop, get approval. Never a
whole page up front.
