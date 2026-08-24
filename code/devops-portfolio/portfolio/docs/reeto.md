# Reeto — Multi-tenant AI Content Platform

**Stack:** Next.js (Vercel) · Supabase (Postgres, Auth, Storage, RLS) · Google Cloud Run · Cloud Scheduler · Cloud Logging · Docker · Playwright · ffmpeg · Gemini API · Anthropic Claude API · Stripe
**Role:** Sole builder

> **Status:** In active development. The render pipeline is proven end to end on Cloud Run; the wider product runs in a dev environment while the UI is finished, and Stripe runs in test mode. Everything below is what is built and how it works.

---

## What it is

Reeto generates on-brand social content for small businesses. The guiding principle is human-in-the-loop: the pipeline proposes, the creator approves. Nothing an AI stage produces reaches a customer's feed until the business owner has reviewed it, whether the run was triggered manually or by the scheduled operator below.

---

## A three-stage AI pipeline with an independent critic

Three named stages, each doing what it is best at, with a person deciding what ships:

- **Strategist (Gemini)** turns the interview answers and brand config into an angle, story arc, and layout, using search grounding for trend signal.
- **Copywriter (Claude)** drafts slide copy within hard constraints: 30 words per slide, one idea per slide, never reuse an input fact verbatim as a headline, at least one slide must reframe rather than narrate.
- **Creative Director (Gemini)** reviews the finished copy against the brand config and layout constraints, flagging off-brand language, contrast failures, restatement, and overflow. In strict mode it can reject and force a retry.

Splitting the writer and the reviewer across two different model families (Claude writes, Gemini judges) makes the review a genuine second opinion instead of a model checking its own work. A human approval step sits after all three.

---

## Deterministic rendering, not generative

The core architectural bet. Instead of generating images with a diffusion model and hoping the text lands, I render HTML through headless Chromium (Playwright) and compose video with ffmpeg, in a Docker container on Cloud Run. Same brand config in, same on-brand result out, every time, at a fraction of a cent per asset. It cannot hallucinate a layout.

---

## Layouts as data, with an automated quality gate

Every layout is a JSON skeleton whose frame and text-slot positions are expressed as percentages of the slide, so it is resolution- and aspect-independent. Adding a layout means adding a config object, not writing a new template file.

From roughly 20 skeletons and about 30 hand-authored pieces (skeletons, type treatments, colour arrangements, background textures), the library composes around 1,500 machine-validated design combinations. Every combination must pass an automated gate before entering the library: contrast ratio at least 4.5:1, no text overflow beyond a slot's max lines, no element collision, and text area no more than 55% of the slide. That gate is the automated stand-in for a human design team.

---

## Multi-tenant isolation

Supabase Postgres with row-level security enforcing organisation-level isolation, plus role-based access and isolation tests, so one tenant cannot read another's data even if application code fails.

---

## Autonomous operator

A Cloud Scheduler job runs the three-stage pipeline weekly and unattended for every active brand, so a fresh proposal is always waiting. Every stage writes structured output to Cloud Logging, so an AI decision can be inspected rather than trusted blindly. Nothing from an unattended run is used until the owner reviews it.

---

## Payments

Stripe billing (test mode), gated on brand count rather than features, so every paid tier gets the full product.

---

## Engineering lessons

- **Constraints beat instructions.** "Write good copy" produces nothing useful; "30 words per slide, one idea per slide, never reuse an input fact as a headline, at least one slide must reframe" produces something a person would actually post.
- **Every stage must make a decision nothing else can.** A stage that only relays or rephrases adds latency and API spend without adding a real decision. That test cut the pipeline down to three stages.
- **Percentage layouts did not survive aspect-ratio changes.** Layouts authored at 4:5 crushed at 16:9; the fix was per-ratio role variants and disabling ratios a skeleton was never designed for.
- **Avoiding the plagiarism trap.** A reference image is analysed for abstract principles only (spacing rhythm, type hierarchy, alignment logic), then discarded; a new structure is composed from those parameters in code, never from the image. Extract the principles, never reproduce the artifact.
