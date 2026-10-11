## What it does

`implement-spec-agentteam` takes a [spec](https://www.aihero.dev/ai-coding-dictionary/spec) and its [tickets](https://www.aihero.dev/ai-coding-dictionary/ticket) and lands the whole thing in one run, the way [implement-spec](https://aihero.dev/skills-implement-spec) does, but every worker is a durable **teammate** rather than a fire-and-forget [sub-agent](https://www.aihero.dev/ai-coding-dictionary/subagent). You can list who is running, wait on one member without blocking the rest, wake a member that has gone quiet with a message, and interrupt a member that has gone sideways.

That listability is the defining constraint. On the sibling skill a worker that goes wrong is a worker you abandon: you cannot see it, steer it, or stop it, so the only recovery is to spawn another and repeat the work. Here the orchestrator steers the worker that is already running, and the ticket graph lives on a **shared task board** that every member reads, so no one has to hold the frontier in their head.

## When to reach for it

You invoke this by typing `/implement-spec-agentteam`, and the agent won't reach for it on its own.

Reach for it whenever the Agent Teams tools are available in your [harness](https://www.aihero.dev/ai-coding-dictionary/harness). Reach for [implement-spec](https://aihero.dev/skills-implement-spec) when they are not: the two skills run the same task graph and land on the same kind of integration branch, and the one you want is decided by whether your harness can keep durable teammates.

| Your situation | Reach for |
| --- | --- |
| Your harness has Agent Teams, and a spec split into tickets with blocking edges | `/implement-spec-agentteam` |
| Your harness has no Agent Teams | [implement-spec](https://aihero.dev/skills-implement-spec) |
| One ticket at a time, in your own [context window](https://www.aihero.dev/ai-coding-dictionary/context-window) | [implement](https://aihero.dev/skills-implement) |
| A spec that isn't split into tickets yet | [to-tickets](https://aihero.dev/skills-to-tickets) first |

## Prerequisites

- **An issue tracker.** The skill reads the tickets from, and resolves them on, the tracker [setup-matt-pocock-skills](https://aihero.dev/skills-setup-matt-pocock-skills) configured. If none has been configured, it stops and tells you to run that first rather than guessing.
- **Tickets with blocking edges**, as [to-tickets](https://aihero.dev/skills-to-tickets) writes them. Without edges the graph is flat and every ticket starts at once.
- **A harness with Agent Teams enabled.** Every worker being a teammate is the whole point, so on a harness without it this skill has nothing to run.

## The exploration pass gates the run

The optional exploration step is optional to *schedule*, not to *wait for*. If you take it, no implementer starts until the explorer's notes exist, because those notes are the thing the implementers read. A run that spawns both at once hands the implementers a note path that does not resolve yet, and every implementer quietly pays to rediscover what was already found.

## One member per ticket, and one merger for the run

Each ticket gets its own teammate, its own git worktree, and its own branch. The merger is different: you create it once at the start and hand it every merge for the whole run, so merge standards stay consistent and no merge waits on a member being spawned.

A teammate reporting done is not a ticket landed. The board task completes when the merger has merged the branch, because the merged state is the only state that counts.

## Working around the roster limit

Teammate slots are never freed. A member that finished, one you interrupted, and even one that failed to create all keep their slot, so the roster is a budget you spend once, at the start. Size it before you staff anything: every ticket in flight, plus the merger, plus whoever reviews at the end.

When a run has more tickets than slots, staff a batch, and let those teammates take the next batch as they finish. The alternative, one new teammate per ticket, stops dead the moment the tickets outnumber the slots.

## Common questions

**A ticket's tests passed in its worktree, but it silently skipped the only test that mattered.**

A worktree is a complete *checkout*, not a complete *environment*. Anything the repo deliberately keeps untracked does not come with it: gitignored fixtures, local databases, large corpora, credentials. One user hit exactly this, and the test was written to skip itself when its corpus was absent, which is the right behaviour for a clean checkout and CI. The ticket whose whole purpose was to re-read that corpus would have gone green having tested nothing.

Give the implementer the escape hatch when you read the ticket: a ticket whose verification depends on untracked material runs in the main checkout, not a worktree. Isolation is the default worth having; it just cannot be unconditional.

**Blocked tickets never unblock, even after their blocker merged.**

Against a tracker whose tickets close when the pull request merges, the blocked-by count stays up for the whole run, because the run ends at the merge. On a four-ticket graph one orchestrator watched two tickets sit blocked the entire time though their blocker had merged early.

The board is what fixes this. The tracker supplies the starting graph; from then on the frontier comes from the board, where a ticket is done when its merger lands it. That is also why the skill has the merger complete the board task rather than the implementer.

**The review loop kept going for hours, and kept finding things.**

This is the known shape of an unbounded review loop: the fix commit changes the diff, the changed diff invites another broad review, and the loop finds something every time. The skill runs one broad review and sends its findings to one implementer. After that, check the findings that were raised; do not start another broad review unless you ask for one.

**Some implementers collided on the same file.**

Worktrees postpone collisions to merge time rather than preventing them. A blocking edge written from ticket text is a guess about which files a ticket touches, and two tickets on "different parts of the codebase" still share a string catalogue or a config registry. When two frontier tickets share a file, either add a blocking edge so they run one after the other, or have the exploration notes fix the exact names each ticket adds. This is the same advice as its sibling skill, and it is the reason the exploration pass is worth taking.

**Cleanup failed and left every worktree behind.**

A compound `git worktree remove … && git branch -d …` dies as a whole when the second half is refused by a branch guard, so the first half never runs either. Run each removal as its own command. Never force a deletion: once integration history has been rewritten (cherry-picked for a linear pull request, or amended after review), the implementer branch is no longer an ancestor, so `-d` refuses it, and discarding that branch is the user's call, not the skill's.

## It's working if

- Every teammate you need is visible in one list, and you can say which ticket each one holds.
- You steer a teammate that went wrong instead of respawning it, and its history comes with it.
- The number of members you created is the number you budgeted at the start, not one more per ticket.
- The frontier comes from the board, so a ticket unblocks the moment its merge lands rather than at the end of the run.
- No implementer started before the exploration notes existed.

## Where it fits

The Agent Teams half of the implementation flow: the pick when your harness has durable teammates, with [implement-spec](https://aihero.dev/skills-implement-spec) as its plain-sub-agent sibling. It closes out on the integration branch with [code-review-agentteam](https://aihero.dev/skills-code-review-agentteam), which runs the two review axes as teammates.

[ask-matt](https://aihero.dev/skills-ask-matt) routes over the whole set.
