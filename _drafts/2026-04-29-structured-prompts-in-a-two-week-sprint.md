---
layout: post
title: "Structured prompts in a two-week sprint"
seo_description: "Where do structured prompts fit in a Scrum process? A practical guide to slicing features into prompt-sized stories, the mini-waterfall failure mode, and the file layout that keeps prompts and code in sync — with examples from ClearRank."
description: "Where do structured prompts fit in the process your team already runs? A practical look at slicing features into prompt-sized stories, avoiding mini-waterfall, and the file layout that's working for me on ClearRank."
permalink: structured-prompts-in-a-two-week-sprint
date: 2026-04-29 11:00:00 +0200
image:
tags: [ai, prompt-engineering, agile, scrum, software-design, clearrank]
---

[FILL: opening anecdote — a real moment of almost-failure or actual failure on a recent ClearRank sprint. The credibility anchor.]

A few weeks ago I [wrote about why prompts belong in version control]({{ site.baseurl }}/prompts-as-architecture-artifacts). The follow-up question I keep getting is the practical one:

*Where do these prompts fit in the process my team already runs?*

Most teams aren't starting from zero. They have Scrum, two-week sprints, Jira tickets, backlog refinement, and a PR review ritual that already works. SPDD has to fit into that, not replace it.

Here's the shape that's working for me on ClearRank.

## The flow

> PRD → Epic → User story → Structured prompt → Code + tests → PR review

The PRD says *why*. The story defines a small deliverable. The prompt translates the story into implementation intent the AI can act on. The code and tests prove it works.

Nothing here removes existing artifacts. The prompt slots between the story and the implementation, and it's the only new thing.

## The failure mode: mini-waterfall

The risk with any spec-shaped artifact is that it slides back into waterfall. SPDD is no exception.

If a team writes one giant prompt at the start of an epic and generates one giant PR at the end, they've made things worse, not better. The prompt becomes a 2,000-line specification, the AI generates a 4,000-line diff, and the reviewer has no realistic way to tell whether the implementation matches the intent.

I came close to doing this on the drift detector. The first version of the prompt tried to cover ingestion, scoring, and reporting in one file.

[FILL: what happened — the moment you realized the prompt had outgrown the story and what you did about it.]

The version that works is small.

## Slicing a feature into prompt-sized stories

The ClearRank Narrative Drift Detector is one feature on the roadmap. It's also seven stories in the backlog:

1. Store the intended brand narrative
2. Collect AI answers from multiple search tools
3. Extract how the brand is described in each answer
4. Compare descriptions against the intended narrative
5. Calculate an explainable drift score
6. Show the drift report
7. Generate content recommendations

Each story is a single sprint item. Each one gets its own REASONS prompt. None of them are larger than the story itself.

This is the unit that works:

- one story
- one focused prompt
- one reviewable PR
- one set of tests

The prompt for story #5 (the scoring) doesn't need to know how ingestion works. It needs to know what `AIAnswer` looks like as input and what `DriftScore` looks like as output. The shared entities live in one file the prompts reference; everything else is local to the story.

## File layout

This is the part nobody else writing about SPDD has answered, so here's what I landed on:

```
docs/prompts/
  _shared/
    entities.md          # BrandProfile, IntendedNarrative, AIAnswer, etc.
    norms.md             # naming, logging, testing standards
    safeguards.md        # global non-negotiables
  drift-detector/
    01-store-narrative.md
    02-collect-answers.md
    03-extract-descriptions.md
    04-compare.md
    05-score.md
    06-report.md
    07-recommendations.md
```

Each story-level prompt references the shared files instead of restating them. When the entity model changes, I update one place. When a story's intent changes, I update one place.

The folder name maps 1:1 to the epic. The file name maps 1:1 to the story ID. A reviewer opening a PR can find the prompt in two clicks.

## What changes in code review

The cadence doesn't change. The artifact list does.

A reviewer is now looking at four things instead of one:

- Does the ticket describe the right problem?
- Does the prompt translate the ticket correctly?
- Does the code follow the prompt?
- Do the tests prove the behavior?

> We don't only review code anymore. We review the chain of intent.

In practice, this is faster, not slower. When the prompt is good, the code review is mostly mechanical — the design conversation already happened on the prompt. When the prompt is bad, the review catches it before 400 lines of code get written on top of a wrong assumption.

[FILL: one real moment from a recent ClearRank PR where the prompt caught something the code review wouldn't have.]

## What SPDD doesn't fix

Story slicing is still hard. Picking the right entities is still hard. Knowing when to write a test is still hard.

SPDD makes implementation intent visible. It doesn't make product thinking automatic. If the story is wrong, the prompt will be wrong, and the code will be wrong — just faster.

The win is narrower than people sometimes claim: the design that used to live in someone's head, or in a closed ChatGPT tab, now lives in the repo where the rest of the team can see it.

For a sprint team shipping AI-assisted code, that's enough.

---

*ClearRank tracks how AI search tools describe your company and flags when that description drifts from your intended positioning. [FILL: link / beta / DM for early access].*
