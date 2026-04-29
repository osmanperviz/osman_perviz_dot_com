---
layout: post
title: "Prompts as architecture artifact"
seo_description: "Structured-Prompt-Driven Development (SPDD) gave a name to a habit I had already been building on ClearRank: treating prompts as version-controlled architecture artifacts. A senior engineer's take on REASONS, prompt/code drift, and why the prompt is where most design thinking now happens."
description: "Structured-Prompt-Driven Development gave a name to a habit I had already been building: treating prompts as version-controlled architecture artifacts. The prompt is becoming the place where implementation design gets captured — not the chat tab where it disappears."
permalink: prompts-as-architecture-artifacts
date: 2026-04-29 10:00:00 +0200
image:
tags: [ai, prompt-engineering, architecture, software-design, clearrank]
---

A days ago, Wei Zhang and Jessie Xia from Thoughtworks published [Structured-Prompt-Driven Development](https://martinfowler.com/articles/structured-prompt-driven/) on Martin Fowler's site.

Reading it felt less like learning something completely new and more like finding a name for a habit I had already been moving toward while building [ClearRank](https://clearrank.io).

Structured-Prompt-Driven Development — SPDD, in short — is the practice of treating prompts as version-controlled engineering artifacts, evolved and reviewed alongside the code they shape.

The thesis is simple: prompts should not live only in private chat history. They can become first-class delivery artifacts — kept in sync with the code they influence.

That is the part I find interesting.

Not "AI can write code."

But: the prompt is becoming a place where a lot of implementation design gets captured.

## What I was doing badly before

My early prompts for [ClearRank](https://clearrank.io) looked something like this:

> Build a narrative drift detector. It should compare how the company wants to be positioned with how AI tools describe it.

The code I got back compiled. It even ran.

But later, I could not easily answer basic questions:

- Why did I structure the scoring this way?
- Which edge cases did I intentionally ignore?
- What did I tell the model not to do?
- Which assumptions were part of the original design?

The reasoning lived in a ChatGPT tab, not in the repository.

That is the failure mode SPDD addresses.

Not necessarily "the AI wrote bad code."

The more dangerous case is: the AI wrote plausible code, but the intent behind it disappeared.

## REASONS, briefly

The article's main contribution is the REASONS Canvas — a structure for prompts covering:

- Requirements
- Entities
- Approach
- Structure
- Operations
- Norms
- Safeguards

Read the article for the full breakdown; it is worth it.

What clicked for me was not only the seven sections. It was the framing.

A prompt that includes domain entities, architecture boundaries, implementation norms, and explicit safeguards is no longer just a request.

It becomes a small design document.

## A ClearRank example

[ClearRank](https://clearrank.io) is an AI visibility product. It tracks how generative search tools describe a company and helps detect when that description drifts away from the company's intended positioning.

One feature where this matters is a narrative drift detector.

The goal is not only to check whether a brand is mentioned. The more important question is:

> Is the AI describing the company in the way the company wants to be understood?

For example, a company may want to be positioned as:

> "An AI visibility platform for tracking brand presence in generative answers."

But AI tools may describe it as:

> "An SEO tool."

That sounds close, but strategically it is different.

The category has shifted. The narrative has drifted.

A weak prompt would be:

> Build a narrative drift detector.

A more useful structured prompt looks like this:

```text
Requirements:
  Compare a company's intended positioning with how AI systems describe it.
  Flag drift when AI answers describe the company too generically,
  inaccurately, or in a strategically different category.

Entities:
  BrandProfile, IntendedNarrative, AIAnswer, Citation, Mention,
  DriftScore, Recommendation

Approach:
  Separate factual errors from positioning drift.
  Treat synonyms as equivalence, not drift.
  Score must be explainable — every drift event cites the source span.

Structure:
  Scoring logic must not depend on the answer-collection layer.
  Raw AI answers are stored separately from interpreted analysis.
  Scoring model is replaceable.

Operations:
  Ingest answers from multiple AI search tools.
  Extract how the brand is described.
  Compare against intended narrative.
  Detect missing concepts, wrong associations, and competitor confusion.
  Produce explainable drift score and content recommendations.

Norms:
  Use domain language only.
  Avoid provider-specific terms in core logic.
  Tests must read like product behavior, not implementation details.

Safeguards:
  Do not turn this into a generic SEO tool.
  Do not treat synonyms as drift.
  Do not mix collection with analysis.
  Do not generate vague recommendations without citing the source span.
```

This is not a longer prompt for the sake of being longer.

Each section answers a question that would otherwise live in my head: what counts as drift, where the boundaries are, which assumptions the model should respect, and which mistakes are not negotiable.

The Safeguards section is the part I underestimated at first.

Without it, the output tended to drift toward generic SEO language — because that is what most of the training data is full of. The safeguards are how I keep the product from regressing into something it was never supposed to be.

## The trade-off nobody talks about

There is a cost to this approach that the article touches on but does not dwell on: prompts drift from code, silently, the same way comments do.

You ship a feature with a clean prompt. Three weeks later, someone patches a bug directly in the code. The prompt no longer reflects what the code does.

Now there are two sources of truth and neither is fully trustworthy.

The article's rule — *when reality diverges, fix the prompt first, then update the code* — is the right discipline. But it is a discipline. Like keeping tests green, it does not enforce itself.

The rule I am moving toward is simple: if a prompt-backed feature gets a bug fix, the prompt gets updated in the same change.

It is annoying.

It is also the only way I have found to keep the prompts from rotting.

## When this is worth it

Not every change deserves a REASONS Canvas. A throwaway script, a CSS tweak, a one-off migration — none of these need a spec.

The places where it earns its weight are domain-heavy: scoring logic, billing, permissions, recommendation engines, AI visibility analysis. Anywhere the cost of a *wrong abstraction* is higher than the cost of *slow coding*.

[ClearRank](https://clearrank.io) is mostly that kind of work, which is why the overhead pays for itself.

If most of your AI-assisted changes are UI glue, SPDD will feel like bureaucracy.

If they are touching the core domain model, it will feel like the cheapest review process you have.

The benefit also compounds when someone else reviews the work. They can see the intent, not only the diff. The prompt becomes the answer to *what was supposed to happen here* — without anyone having to dig through git blame to find out.

## What I am doing next

I plan to publish a few real examples from the ClearRank prompt folder — the drift detector, the visibility scoring engine, and the citation analyzer — once they are cleaned up. Each will include the REASONS structure and the commit history that shows how it evolved.

The point is not to show off prompts.

The point is to show that the design conversation can live in the same place as the code, in the same review process, with the same rigor.

I am not just using AI to write code faster. I am adapting my engineering workflow so AI-assisted work remains reviewable, versioned, and aligned with product intent.

If you are building something AI-assisted and the code keeps drifting from what you actually meant, the fix is probably not a better prompt.

It is treating the prompt like code: reviewed, versioned, and owned.

---

*[ClearRank](https://clearrank.io) tracks how AI search tools describe your company and flags when that description drifts from your intended positioning. Reach out if you want to follow along *
