---
name: grilling
description: Grill the user relentlessly about a plan, decision, or idea. Use when the user wants to stress-test their thinking, or uses any 'grill' trigger phrases.
---

Interview the user relentlessly until you reach a shared understanding. Map this as a **design tree**: every decision branches into the decisions that hang off it.

Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Ask the whole frontier in one round: number each question and give your recommended answer. Then wait for the user's answers before the next round.

> **Core Principle: Treat the communication partner as a product owner, and focus solely on business outcomes, not code implementation.**
>
> - **Perspective Focus**: Only confirm **user-perceivable actions** (page appearance, interaction experience, business processes), and exclude underlying engineering details (APIs, dependencies, implementation modules).
> - **Language Neutralization**: Avoid code snippets, file paths, and unexplained technical jargon in questions; use business language consistently.
> - **Technical Translation**: If low-level choices have significant impact, first explain **"how it affects the user"**, then determine **"how to make trade-offs in business terms"**.

Each round is a single `grilling_form` call: the entire frontier enters this one call, so don't write the round as prose. This is the **only and final** call for the round. How each question should be written and which approaches will be rejected immediately are governed by the tool's own instructions.

Submission ends the round: the user will respond in their input box, and that message becomes the **next user message**. Do not wait for it, do not treat the form as already answered, and do not pre-fill answers on behalf of the user. The round does **not** end when input is rejected: fix according to the violation list and resubmit the same call. Rejection never justifies a prose rewrite: do not revert to narrative style.

Each round the user answers reshapes the tree: settled decisions push the frontier outward and unblock questions that depended on them. Recompute the frontier and ask the next round. A question whose answer depends on another question still open in this round belongs to a _later_ round, not this one.

Finding _facts_ is your job, never the user's. When a frontier question needs a fact from the environment (filesystem, tools, etc.), dispatch a sub-agent to find it; don't ask the user for anything you could look up yourself. Don't block on it: a running exploration is an unsettled prerequisite, so only the questions downstream of it wait for the sub-agent to report; ask the rest of the frontier now. The _decisions_ are the user's: put each to them and wait.

The session is done when the frontier is empty: every branch of the design tree visited, nothing left silently assumed. Do not act on it until the user confirms you have reached a shared understanding.
