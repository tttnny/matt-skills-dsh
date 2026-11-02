---
name: code-review-agentteam
description: "Review the changes since a fixed point (commit, branch, tag, or merge-base) along two axes, Standards (does the code follow this repo's documented coding standards?) and Spec (does the code match what the originating issue/spec asked for?), with the two reviews run as DSH Agent Teams teammates. Use when the user wants to review a branch, a PR, work-in-progress changes, or asks to \"review since X\"."
---

Two-axis review of the diff between `HEAD` and a fixed point the user supplies:

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the originating issue / spec?

Each axis runs in its own context, which is what keeps the two reports independent. This skill is the Lead's to run: every member gets the same team tools, but `spawn_teammate` and `interrupt_agent` refuse a teammate, so a teammate that needs this review asks the Lead for it.

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`.

## Process

### 1. Pin the fixed point and freeze the diff

Whatever the user said is the fixed point (a commit SHA, branch name, tag, `main`, `HEAD~5`, etc.). If they didn't specify one, ask for it.

Confirm the fixed point resolves (`git rev-parse <fixed-point>`), the diff is non-empty, and the tree is clean (`git status --porcelain`). A bad ref, an empty diff, or a dirty tree fails here, not inside two teammates.

The reviewers can write to your checkout, so the diff can move while the reviews run. Freeze it once: write `git diff <fixed-point>...HEAD` to a patch file under a git-ignored path or outside the checkout, and record the diff hash. Hand both reviewers the artifact, never a command that re-reads a live tree.

Done when the patch path, the commit list (`git log <fixed-point>..HEAD --oneline`), and the diff hash are in hand.

### 2. Identify the spec source

Look for the originating spec, in this order:

1. Issue references in the commit messages (`#123`, `Closes #45`, GitLab `!67`, etc.), fetched via the workflow in the tracker doc.
2. A path the user passed as an argument.
3. A spec file under `docs/`, `specs/`, or `.scratch/` matching the branch name or feature.
4. If nothing is found, ask the user where the spec is.

Done when you have the spec's contents or a clear answer that there is none.

### 3. Identify the standards sources

Search the repo for every file that documents how code should be written. When `CODING_STANDARDS.md` or `CONTRIBUTING.md` exists, it must be on the list.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** below: a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation. Like any standard here, skip anything tooling already enforces.

Each smell reads *what it is* → *how to fix*; match it against the diff:

- **Mysterious Name**: a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code**: the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps**: the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches**: the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change**: one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality**: abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

Done when the standards list is fixed and you have the baseline in hand to paste into the brief.

### 4. Spawn the two reviewer teammates

Create one teammate per axis with `spawn_teammate`, each with a complete, self-contained task. Name them so the axis is obvious (`review-standards`, `review-spec`).

Pass `context: "fresh"` for both. A fork inherits your completed turns, including your own reading of the diff and any finding you have already formed, which is the cross-contamination the two axes exist to prevent.

The team cap counts every teammate the team has ever created, failed creations included, and no name is ever reused, so a full team never makes room again. When the team has no room for both reviewers, run the axes one after the other in your own context and say in the report that they ran unisolated. Check your tool list before reaching for `subagent`: the Agent Teams composition disables it, and a subagent is not a member, so `list_agents` and `interrupt_agent` never reach it.

A spawn that throws leaves a durable `failed` row when the provider failed: `list_agents` shows it with its error in `diagnostics`, its name is burned, and it still counts against the cap. A `Team member limit <n> reached` failure leaves no row at all. Retry only the axis that is missing, under a new name when its row is `failed`.

**Both reviewers are read-only.** They hold `edit` and `write`, so add to each brief: review only, leave the tree alone.

**Standards teammate's task** carries:

- the frozen patch path, and the commit list;
- the list of standards-source files you found in step 3, plus the smell baseline from step 3 pasted in full; the teammate gets none of your conversation, so anything it needs must be in the brief;
- the brief: "Open your report with `Standards:`. Report, per file/hunk where relevant, (a) every place the diff violates a documented standard: cite the standard (file + the rule); and (b) any baseline smell you spot: name it and quote the hunk. Distinguish hard violations from judgement calls: documented-standard breaches can be hard, baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. Under 400 words."
- an instruction to end its turn with the findings as its closing message.

**Spec teammate's task** carries:

- the frozen patch path, and the commit list;
- the path or fetched contents of the spec;
- the brief: "Open your report with `Spec:`. Report: (a) requirements the spec asked for that are missing or partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. Under 400 words."
- an instruction to end its turn with the findings as its closing message.

If the spec is missing, spawn only the Standards teammate and record the missing Spec axis in the final report.

Done when every axis you are running has either a spawned teammate (`member.target` with `status` `running` or `inactive`) or a recorded reason it ran unisolated, and each teammate holds the brief above. `inactive` on return only means no turn is executing; it is not a spawn failure and not proof the review finished.

### 5. Collect

Each reviewer's closing message is delivered to you automatically as a settlement notice when its turn ends; it is not a tool result. The notice names the child by session id, which your tools never show, so the axis label at the top of each report is what tells the two apart. An empty closing message arrives as "It left no closing message."

If an active reviewer ends without a report, send it a message asking for the findings. A `failed` or `provisioning` member cannot be messaged at all, so spawn a replacement under a new name or run that axis yourself.

Done when both reports are in hand, or you have recorded which axis produced none and why.

### 6. Aggregate

Present the two reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned, so neither axis can mask the other (see _Why two axes_).

Before writing, re-check `git status --porcelain` and the diff hash against step 1. If either moved, the reviews describe a tree that no longer exists: say so at the top of the report.

End with a one-line summary: total findings per axis, and the worst issue _within each axis_ (if any). Each axis gets its own worst issue, which is what keeps the separation from collapsing into a single ranking.

Done when both headings are present, the tree check is recorded, and the summary line is written.

## Why two axes

A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the issue asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.
