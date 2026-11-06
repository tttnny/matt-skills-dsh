---
name: code-review-agentteam
description: "Review the changes since a fixed point (commit, branch, tag, or merge-base) along two axes, Standards (does the code follow this repo's documented coding standards?) and Spec (does the code match what the originating issue/spec asked for?), with each axis run as a named DSH subagent you can message and interrupt. Use when the user wants to review a branch, a PR, work-in-progress changes, or asks to \"review since X\"."
---

Two-axis review of the diff between a fixed point and the current commit:

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the originating issue / spec?

Each axis runs in its own subagent context, which is what keeps the two reports independent. Each reviewer is a **named** child, so you can ask it a follow-up without paying for the reading twice, and stop it when it has gone past the diff.

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`.

## Steps

### 1. Pin the range

Resolve both ends and keep them: `git rev-parse <fixed-point>` and `git rev-parse HEAD`. If the user named no fixed point, ask for one.

Hand out the range, never a moving target. Reviewers get `git diff <fixed>...<head>`, with both SHAs spelled out, so anything you do to the working tree afterwards cannot change what they read. Confirm the range is non-empty before spending two children on it.

Done when you hold two SHAs, a non-empty diff, and the commit list (`git log <fixed>..<head> --oneline`).

### 2. Find the spec and the standards

Look for the originating spec: issue references in the commit messages, a path the user passed, or a spec file under `docs/`, `specs/` or `.scratch/` matching the branch. If there is none, say so, and the Spec axis reviews the commits' stated intent instead.

Then collect the standards sources: every file in the repo that documents how code should be written. When `CODING_STANDARDS.md` or `CONTRIBUTING.md` exists, it is on the list. Add this smell baseline when the repo documents none:

- **Divergent Change**: one module changes for unrelated reasons. → split it.
- **Shotgun Surgery**: one small change edits many files. → move the parts together.
- **Feature Envy**: a function uses another object's data more than its own. → move it there.
- **Data Clumps**: the same two or three fields travel together. → give them a type.
- **Primitive Obsession**: an int or string stands in for a domain concept. → a small type.
- **Speculative Generality**: hooks and parameters nothing calls yet. → delete them.
- **Temporary Field**: a field that only holds a value in some cases. → a narrower object.
- **Refused Bequest**: a subclass ignores most of what it inherits. → composition.

Done when the spec text (or the finding that there is none) and the standards list are in hand.

### 3. Spawn both reviewers in one message

Two `subagent` calls, issued together, each named for its axis: `standards-reviewer` and `spec-reviewer`.

Use `subagent`, not `subagent_fork`. A fork inherits your completed turns, including your own reading of the diff and any suspicion you already formed, which is the contamination the two axes exist to prevent.

Both briefs are self-contained, because a fresh child sees none of this conversation:

- **both**: the two SHAs, the diff command, the commit list, `review only, do not modify the working tree or create commits` (they hold `edit` and `write`), and the report shape: open with the axis name, then findings, each naming its file, the hunk, and the rule or spec line it breaks; say "no findings" rather than inventing one;
- **Standards**: the standards file list and the smell baseline, pasted in full, plus the instruction to separate documented-standard breaches (hard) from smell judgements, and to skip anything the repo's tooling already enforces;
- **Spec**: the spec text (or the statement that there is none), plus the instruction to report what the spec asked for and the diff does not do, what the diff does and the spec never asked for, and where it departs from the spec's intent.

Then stop working. Each completion notice starts a new turn for you.

Done when both children have been asked for their report.

### 4. Collect, and steer only if you must

Keep the two reports apart. A reviewer that wandered (started editing, wrote an essay about the codebase, re-read the whole repo) is cheaper to redirect than to replace: `interrupt_agent`, then `send_message` with the correction. It keeps everything it already read.

Ask follow-ups of the same child when a finding is unclear. It has the reading in context; a new child repeats it from scratch.

If a spawn fails, retry that axis once. If it fails again, run that axis yourself and say in the report which axis ran in your context rather than its own.

Done when you hold two reports.

### 5. Present

Under two headings, verbatim findings, never merged or re-ranked: a change can pass one axis and fail the other, and running them under one summary is how one axis masks the other. Close with a short verdict: mergeable, or the findings that must be fixed first.

If a finding is a judgement call rather than a documented breach, say which it is; the caller is the one who decides.

## Tracking the review on the board

When the session has Agent Teams on and this review is part of a team run, put it on the board so it is visible next to the work it reviews: one task per axis with `team_task_create` (`write_scopes` empty, the reviewers write nothing), the axis in the subject, and both tasks unblocked. Each reviewer's brief then carries its task id and the instruction to claim it with `team_task_get` then `team_task_update({ action: 'claim' })` before starting. Complete each task with `team_task_update({ action: 'complete', expected_revision })` when its report lands.

Close the review's tasks before presenting; a review left open on the board reads as work still running.
