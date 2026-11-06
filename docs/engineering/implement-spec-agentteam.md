## What it does

`implement-spec-agentteam` takes a [spec](https://www.aihero.dev/ai-coding-dictionary/spec) and its [tickets](https://www.aihero.dev/ai-coding-dictionary/ticket) and lands the whole thing in one run, the way [implement-spec](https://aihero.dev/skills-implement-spec) does. The difference is where the plan lives and what a worker is: the ticket graph is copied onto the shared **task board**, and every worker is a **named subagent** of the Lead that can be messaged, interrupted, and asked follow-ups.

The board is the defining constraint. It holds the blockers, so the frontier is read from it rather than remembered; it holds advisory write scopes, so two tickets that would collide are visible before either starts; and it holds the owner, so a ticket whose implementer died is not lost. A ticket is complete when its merge landed, which is what unblocks the next ticket the moment the merge happens rather than at the end of the run.

## When to reach for it

You invoke this by typing `/implement-spec-agentteam`, and the agent won't reach for it on its own.

Reach for it when Agent Teams is on for the session, which you can tell from your tool list: the board tools are `team_task_create`, `team_task_get`, `team_task_list` and `team_task_update`. If they are missing, Agent Teams is off for this session and [implement-spec](https://aihero.dev/skills-implement-spec) runs the same graph without a board.

| Your situation | Reach for |
| --- | --- |
| Agent Teams is on, and the spec is split into tickets with blocking edges | `/implement-spec-agentteam` |
| Agent Teams is off for this session | [implement-spec](https://aihero.dev/skills-implement-spec) |
| One ticket at a time, in your own [context window](https://www.aihero.dev/ai-coding-dictionary/context-window) | [implement](https://aihero.dev/skills-implement) |
| A spec that isn't split into tickets yet | [to-tickets](https://aihero.dev/skills-to-tickets) first |

## Prerequisites

- **An issue tracker.** The skill reads the tickets from, and resolves them on, the tracker [setup-matt-pocock-skills](https://aihero.dev/skills-setup-matt-pocock-skills) configured. If none has been configured, it stops and tells you to run that first rather than guessing.
- **Tickets with blocking edges**, as [to-tickets](https://aihero.dev/skills-to-tickets) writes them. Without edges the graph is flat and every ticket starts at once.
- **Agent Teams on for the session.** The board tools only exist while it is, and the Lead cannot turn it on itself.
- **Git worktrees.** Every implementer works in its own checkout, cut from the integration branch tip when its ticket joins the frontier.

## The exploration pass gates the run

The optional exploration step is optional to *schedule*, not to *wait for*. If you take it, no implementer starts until the explorer's notes exist, because those notes are the thing the implementers read. A run that spawns both at once hands the implementers a note path that does not resolve yet, and every implementer quietly pays to rediscover what was already found.

The notes live in the integration checkout, under a path that checkout ignores, and the explorer reports the absolute path. Later implementers work in their own worktrees, where the same relative path would resolve to the wrong tree, so the absolute path is what makes the notes readable from every one of them.

## One worker per ticket, and the Lead does every merge

Each ticket gets a worktree cut from the integration branch tip of the moment it joins the frontier, and the [subagent](https://www.aihero.dev/ai-coding-dictionary/subagent) is spawned with that path as its working directory. The path is handed over in the spawn call itself, so the isolation is real rather than nominal: a brief that only prefixes its shell commands moves `bash` and leaves `read`, `edit` and `write` pointing at the Lead's checkout.

A blocked ticket holds no worktree. A base cut before its blockers landed is already behind by the time it starts, and turns a merge that should be a fast-forward into a conflict.

The Lead is the merger. It merges every finished branch into the integration branch in its own checkout, runs the build and tests there, and only then completes the board task, because a second member running Git in that checkout contends on one index that nothing guards. A report goes to the Lead the moment the implementer sends it, and an implementer reporting done is not a ticket landed: the merged state is the only state that counts.

## The frontier comes from the board

The tracker supplies the starting graph. From then on the frontier is whatever `team_task_list` reports as ready, and a task becomes ready when every task it is blocked by is complete. That is what fixes the shape the tracker cannot: against a tracker whose tickets close when a pull request merges, the blocked-by count stays up for the whole run, so tickets sit blocked even though their blockers landed early.

The board never wakes anyone. A task changing state ends no wait and starts no idle member, so when a member must act on a new task the Lead sends it a message.

## The closing review needs a clean tree

The run ends with [code-review-agentteam](https://aihero.dev/skills-code-review-agentteam) on the integration branch, with the commit the branch was cut from as its fixed point. That is what lets an autonomous run review itself without stopping to ask where to start from.

That review pins its range to two commit SHAs, so a dirty tree does not corrupt the review, but the run still keeps the integration checkout clean by construction: the Lead is the only member who commits there, implementers work in their own worktrees, and a stray file left in the integration checkout is what breaks the final commit.

## Common questions

**A ticket's tests passed in its worktree, but it silently skipped the only test that mattered.**

A worktree is a complete *checkout*, not a complete [environment](https://www.aihero.dev/ai-coding-dictionary/environment). Anything the repo deliberately keeps untracked does not come with it: gitignored fixtures, local databases, large corpora, credentials. One user hit exactly this, and the test was written to skip itself when its corpus was absent, which is the right behaviour for a clean checkout and CI. The ticket whose whole purpose was to re-read that corpus would have gone green having tested nothing.

The fix belongs in the ticket: a ticket whose verification needs untracked material either gets that material committed, or generated by a setup step its worktree can run, or it runs outside this skill, one ticket at a time with [implement](https://aihero.dev/skills-implement). A test that skips itself when its material is missing goes green either way, so the ticket has to say which case it is.

**An implementer's edits landed in the integration checkout.**

It was spawned without a working directory, or its brief only prefixed shell commands. Pass `cwd` in the spawn call and put the path in the brief as well, with the instruction to stay inside it. The prompt is what keeps it there; the working directory is what makes `read`, `edit` and `write` agree with `bash`.

**Blocked tickets never unblocked, even after their blocker merged.**

Either the blocker's task was never completed, or the dependency was written against tracker ids. `blocked_by` takes board task ids. The Lead completes the blocker's task in the same breath as the merge, which is what makes the dependents ready.

**The review loop kept going for hours, and kept finding things.**

This is the known shape of an unbounded review loop: the fix commit changes the diff, the changed diff invites another broad review, and the loop finds something every time. The skill runs one broad review, sends its findings to one implementer, and merges that implementer's branch when it reports. After that, check the findings that were raised; do not start another broad review unless you ask for one.

**Some implementers collided on the same file.**

Worktrees postpone collisions to merge time rather than preventing them. A blocking edge written from ticket text is a guess about which files a ticket touches, and two tickets on "different parts of the codebase" still share a string catalogue or a config registry. The board warns only about tasks already in progress, so at staffing time it says nothing about the batch you are about to start: read the ready tasks' write scopes and start a set that is disjoint from each other and from the work in flight. A collision that survives to the merge is a conflict even when the two worktrees never touched.

**An implementer died mid-ticket. What happens to its task?**

The task stays in progress under the dead member, because ownership is never released automatically. Reassign it to a replacement with `team_task_update({ action: 'reassign', owner })`, which only the Lead can do, and give the replacement a new name: a name is unique among the Lead's children, and the dead one still holds its own. The replacement reads the dead member's branch before starting, so the work is not thrown away.

**Cleanup failed and left every worktree behind.**

A compound `git worktree remove … && git branch -d …` dies as a whole when the second half is refused by a branch guard, so the first half never runs either. Run each removal as its own command. The worktree removal is forced, which is what a checkout with leftover build output needs. A branch that `-d` refuses after integration history was rewritten (cherry-picked for a linear pull request, or amended after review) is the user's call, not the skill's.

## It's working if

- Every running implementer appears in one list, and you can say which board task each one holds.
- The frontier comes from the board, so a ticket unblocks the moment its merge lands rather than at the end of the run.
- A worktree appears when its ticket joins the frontier, cut from the branch tip of that moment, and never sits waiting on a blocker.
- No implementer started before the exploration notes existed.
- Every implementer's `read`, `edit` and `write` resolved inside its own worktree.
- Every merge happened in the Lead's checkout, and every board task closed after it.
- You steer an implementer that went wrong instead of respawning it, and its history comes with it.

## Where it fits

The Agent Teams half of the implementation flow: the pick when Agent Teams is on for the session, with [implement-spec](https://aihero.dev/skills-implement-spec) as its plain-subagent sibling. It closes out on the integration branch with [code-review-agentteam](https://aihero.dev/skills-code-review-agentteam), which runs the two review axes as named subagents.

[ask-matt](https://aihero.dev/skills-ask-matt) routes over the whole set.
