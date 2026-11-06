## What it does

`code-review-agentteam` reviews the diff between `HEAD` and a fixed point you name (a commit, a branch, a tag, `main`, `HEAD~5`) along two axes, the way [code-review](https://aihero.dev/skills-code-review) does. **Standards** asks whether the code follows how this repo writes code. **Spec** asks whether the code does what the originating issue or [spec](https://www.aihero.dev/ai-coding-dictionary/spec) asked for.

Each axis runs as its own [subagent](https://www.aihero.dev/ai-coding-dictionary/subagent), and each one is **named**: `standards-reviewer` and `spec-reviewer`. That name is the whole difference from the plain sibling. You can ask a reviewer a follow-up question while it still has the diff in context, and you can stop one that has wandered past the diff. A review you cannot steer is a review that keeps billing.

The diff is pinned to two commit SHAs, the fixed point and the current `HEAD`, and both reviewers are given that range spelled out. Whatever happens to the working tree afterwards cannot change what they read, which matters when the review runs as the closing step of an [implement-spec-agentteam](https://aihero.dev/skills-implement-spec-agentteam) run.

## When to reach for it

Type `/code-review-agentteam`, or the agent reaches for it automatically when a task fits, which is what lets [implement-spec-agentteam](https://aihero.dev/skills-implement-spec-agentteam) call it as its closing step. Whoever runs it spawns the reviewers, so an agent running inside a team can run it too, subject to its delegation depth.

| Your situation | Reach for |
| --- | --- |
| Reviewing a diff on a [harness](https://www.aihero.dev/ai-coding-dictionary/harness) with Agent Teams | `/code-review-agentteam` |
| Reviewing a diff anywhere else | [code-review](https://aihero.dev/skills-code-review) |
| A single ticket's work before it lands | [implement](https://aihero.dev/skills-implement), which runs both axes per ticket |
| A finding you want explained rather than re-read | `/code-review-agentteam`, where the reviewer is still there to ask |

## Prerequisites

- **An issue tracker.** The Spec axis reads the originating issue from the tracker [setup-matt-pocock-skills](https://aihero.dev/skills-setup-matt-pocock-skills) configured. If none has been configured, it stops and tells you to run that first rather than guessing.
- **A non-empty diff against a ref that resolves.** Both ends are resolved before any reviewer is spawned, so a typo in the fixed point fails immediately instead of inside two children.
- **A checkout the reviewers leave alone.** They hold `edit` and `write` and work in the same directory tree as you, so both briefs say to review only and commit nothing.

## The two axes stay separate

The two reports are presented side by side and are never merged or reranked, because a change can pass one axis and fail the other. Code that follows every standard but implements the wrong thing is a **Standards pass, Spec fail**. Code that does exactly what the issue asked but breaks the project's conventions is a **Spec pass, Standards fail**. Reporting them under their own headings is what stops one axis from masking the other.

## Reviewers are not a rubber stamp

A reviewer is asked to find problems, so it finds them: some are real, some are judgement calls dressed as violations. The Standards axis is explicit about the difference, carrying the repo's documented standards as hard violations and the Fowler smell baseline as heuristics that a documented repo standard can override.

That same discipline is what makes the output usable. A findings list that separates "this breaks a rule the repo wrote down" from "this looks like a Feature Envy to me" is one you can act on without re-reading the whole diff yourself.

## Common questions

**The reviewers wrote a full essay instead of a findings list.**

Both briefs cap the report at the length the findings need and nothing more, and both ask for the finding with its location and the standard or spec line it breaks. Where a reviewer padded, the brief is what to tighten. Once the review is running, `send_message` to the reviewer costs one message and no re-reading.

**One reviewer found nothing, and I don't believe it.**

Ask the reviewer rather than spawning a fresh one: it answers, and a fresh child redoes the reading from scratch. The one thing worth checking first is whether it got the material it was owed. The Standards reviewer needs the standards file list, and the Spec reviewer needs the spec text; if the spec is missing entirely, the skill spawns only the Standards reviewer and says so in the report.

**A spawn failed.**

Retry that axis once: two spawns can partially succeed, so one axis may already be running, and only the missing one is worth re-firing. If it fails again, run that axis in your own context and say so in the report, so the reader knows the two reports were not produced under the same isolation.

**The Standards review flagged things the linter already enforces.**

Its brief says to skip anything tooling enforces, because a finding the linter would have caught costs a reviewer's attention to report and the reader's attention to dismiss. Where it keeps happening, check whether the repo's lint command is actually wired into anything: a rule that no guardrail runs is a rule that reads as a suggestion.

**A reviewer contradicted the other one.**

That is the design, not a failure. The two axes are deliberately separate, so the aggregation never picks a winner between them. Where they overlap on the same line you are looking at a genuine trade-off, and the call is yours.

## It's working if

- Both reviewers exist as named agents while they run, and you can stop either one.
- A follow-up question goes to the reviewer that raised the finding, not to a new one.
- The two reports arrive under their own headings, neither summarising the other.
- Findings you act on name their location and the rule or spec line they break.
- A finding the repo's own tooling already enforces never appears.
- You can tell, from the report alone, which findings are hard violations and which are judgement calls.
- Both reviewers read the same pair of commit SHAs.

## Where it fits

The review half of the [implement-spec-agentteam](https://aihero.dev/skills-implement-spec-agentteam) flow, and a standalone you can point at any branch, pull request, or work in progress whenever you want a second read on a diff. Its plain-sub-agent sibling is [code-review](https://aihero.dev/skills-code-review).

[ask-matt](https://aihero.dev/skills-ask-matt) routes over the whole set.
