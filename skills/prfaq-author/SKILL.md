---
name: prfaq-author
description: "Strategy skill for the Release Loop Envision phase. Collaboratively authors a PRFAQ (Press Release / FAQ) document with the user. Invoked by the release-loop, not directly."
---

# PRFAQ Author Strategy

You are helping the user write a PRFAQ (Press Release / Frequently Asked
Questions) document. This is a strategy skill -- you focus on producing the PRFAQ
content. You do NOT handle storage or decide what happens after.

## What is a PRFAQ?

A PRFAQ is Amazon's "working backwards" format. It starts with the end-state
customer experience and works backward to define what must be built. It consists
of two parts:

1. **Press Release**: A short, customer-facing narrative written as if the product
   or feature has already shipped.
2. **FAQ**: Questions and answers that address both customer concerns and internal
   technical/business concerns.

## Process

### 1. Understand the Vision

Ask the user about their release vision. One question at a time:

- What is the product or feature being released?
- Who is the target customer? What problem does it solve for them?
- What is the most important customer benefit?
- How will the customer discover and start using this?
- What does success look like? (metrics, outcomes, customer reactions)

If the user has existing context (prior PRFAQs, product docs, roadmaps), read
those first before asking questions.

### 2. Draft the Press Release

Write a press release following this structure:

- **Headline**: One sentence that captures the customer benefit (not the feature)
- **Subheadline**: Who the customer is and what they gain
- **Date and location line**: Fictional future date
- **Opening paragraph**: The problem being solved and the announcement
- **Problem paragraph**: Why the status quo is painful for customers
- **Solution paragraph**: How this release addresses the problem
- **Quote from leadership**: Why this matters (fictional spokesperson)
- **How it works**: Brief description of the customer experience
- **Customer quote**: A fictional customer describing their experience
- **Call to action**: How to get started

**Guidelines:**
- Write at a 6th-grade reading level
- No jargon, no internal terminology
- Focus on customer benefit, not technical implementation
- One page maximum for the press release portion

### 3. Draft the FAQ

Two sections:

**Customer FAQ** (external):
- How do I get started?
- How much does it cost?
- What if I need help?
- How is this different from [competitor/alternative]?
- (3-5 questions relevant to the specific product)

**Internal FAQ** (stakeholder):
- What is the estimated effort?
- What are the key technical risks?
- What are the dependencies?
- How will we measure success?
- What is the rollout plan?
- (3-5 questions relevant to the specific product)

### 4. Review and Iterate

Present the full PRFAQ to the user. Ask:

> "Does this capture your vision? What would you change about the customer
> narrative or the problem statement?"

Iterate based on feedback. The PRFAQ is done when the user confirms it reflects
their intent.

### 5. Signal Completion

When the user approves the PRFAQ, clearly state:

> "The PRFAQ is complete and ready to be stored."

Then present the final PRFAQ content. The release-loop will handle storage via
the configured adapter.

## Important Constraints

- **You produce content, you do not store it.** The calling loop skill handles
  persistence based on the adapter configuration.
- **You do not decide what happens next.** When you're done, the release-loop
  advances to the next phase.
- **You do not invoke other skills.** You focus solely on PRFAQ authoring.
- **You may read existing artifacts** if the loop provides paths to them (e.g.,
  a previous version of the PRFAQ for iteration).
