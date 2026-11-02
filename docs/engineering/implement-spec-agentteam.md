## What it does

`implement-spec-agentteam` takes a [spec](https://www.aihero.dev/ai-coding-dictionary/spec) and its [tickets](https://www.aihero.dev/ai-coding-dictionary/ticket) and lands the whole thing in one run, the way [implement-spec](https://aihero.dev/skills-implement-spec) does, but every worker is a durable **teammate** rather than a fire-and-forget [sub-agent](https://www.aihero.dev/ai-coding-dictionary/subagent). You can list who is running, wait on one member without blocking the rest, wake a member that has gone quiet with a message, and interrupt a member that has gone sideways.

That listability is the defining constraint. On the sibling skill a worker that goes wrong is a worker you abandon: you cannot see it, steer it, or stop it, so the only recovery is to spawn another and repeat the work. Here the Lead steers the worker that is already running, and the ticket graph lives on a **shared task board** that every member reads, so no one has to hold the frontier in their head.

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
- **Room in the team.** One teammate per ticket, one more if you take the exploration pass, plus two slots held back for the closing review. A run already at its cap runs the review in one context instead.

## The exploration pass gates the run

The optional exploration step is optional to *schedule*, not to *wait for*. If you take it, no implementer starts until the explorer's notes exist, because those notes are the thing the implementers read. A run that spawns both at once hands the implementers a note path that does not resolve yet, and every implementer quietly pays to rediscover what was already found.

The notes live in the integration checkout, under a path that checkout ignores, and the explorer reports the absolute path. Later teammates work in their own worktrees, where the same relative path would resolve to the wrong tree, so the absolute path is what makes the notes readable from every one of them.

## One member per ticket, and the Lead does every merge

Each ticket gets a teammate, its own git worktree, and its own branch, and the worktree is cut when the ticket joins the frontier, from the integration branch tip of that moment. A blocked ticket holds no worktree: a base cut before its blockers landed is already behind by the time it starts, and turns a merge that should be a fast-forward into a conflict resolution.

A teammate starts in the Lead's directory, so the worktree has to be handed to it: the brief carries the absolute path and says to move there as its first action, which is what makes the checkout isolation real rather than nominal.

The Lead is the merger. It merges every finished branch into the integration branch in its own checkout, runs the build and tests there, and only then completes the board task, because a second member running Git in that checkout contends on one index that nothing guards. A report goes to the Lead the moment the teammate sends it, and a teammate reporting done is not a ticket landed: the merged state is the only state that counts.

## The closing review needs a clean tree

The run ends with [code-review-agentteam](https://aihero.dev/skills-code-review-agentteam) on the integration branch, out of the two slots held back for it, and it is given the commit the integration branch was cut from as its fixed point. That is what lets an autonomous run review itself without stopping to ask where to start from.

That review refuses a dirty tree, and the run keeps the tree clean by construction: the Lead is the only member who commits in the integration checkout, the worktree pool sits in `.git/info/exclude`, and the exploration notes live under a path the checkout ignores. A worker committing in that checkout, or a stray file left in it, is what breaks the closing step.

## The team is a fixed budget

A slot spent on one teammate stays spent, so the run reuses an existing teammate before it spawns a new one: a finished implementer keeps its history, and the next ready ticket goes to it with a message. A new teammate appears only when every existing one is busy.

## Common questions

**A ticket's tests passed in its worktree, but it silently skipped the only test that mattered.**

A worktree is a complete *checkout*, not a complete [environment](https://www.aihero.dev/ai-coding-dictionary/environment). Anything the repo deliberately keeps untracked does not come with it: gitignored fixtures, local databases, large corpora, credentials. One user hit exactly this, and the test was written to skip itself when its corpus was absent, which is the right behaviour for a clean checkout and CI. The ticket whose whole purpose was to re-read that corpus would have gone green having tested nothing.

The run keeps every worker in its own worktree, because the integration checkout has to stay clean for the closing review, so there is no main-checkout branch to fall back on. Catch it when you read the ticket instead: a ticket whose verification needs untracked material either gets that material committed or generated by a setup step the worktree can run, or it runs outside this skill, one ticket at a time with [implement](https://aihero.dev/skills-implement). A test that skips itself when its material is missing goes green either way, so the ticket has to say which case it is.

**A worker's edits landed in the integration checkout, not its worktree.**

A teammate starts in the Lead's directory, so a brief that only prefixes shell commands moves `bash` and leaves `read`, `edit` and `write` pointing at the wrong tree. The brief carries the worktree's absolute path and the working-directory call that moves all four together, and that call is the teammate's first action.

**Blocked tickets never unblock, even after their blocker merged.**

Against a tracker whose tickets close when the pull request merges, the blocked-by count stays up for the whole run, because the run ends at the merge. On a four-ticket graph one run watched two tickets sit blocked the entire time though their blocker had merged early.

The board is what fixes this. The tracker supplies the starting graph; from then on the frontier comes from the board, where a ticket is done once its merge has landed. That is also why the Lead closes the board task at the merge rather than the implementer closing it on report: the merged state is the only state that counts.

**The review loop kept going for hours, and kept finding things.**

This is the known shape of an unbounded review loop: the fix commit changes the diff, the changed diff invites another broad review, and the loop finds something every time. The skill runs one broad review, sends its findings to one implementer, and merges that implementer's branch when it reports. After that, check the findings that were raised; do not start another broad review unless you ask for one.

**Some implementers collided on the same file.**

Worktrees postpone collisions to merge time rather than preventing them. A blocking edge written from ticket text is a guess about which files a ticket touches, and two tickets on "different parts of the codebase" still share a string catalogue or a config registry. The board warns only about tasks that are already in progress, so at staffing time it says nothing about the batch you are about to start: read the ready tasks' write scopes and staff a set that is disjoint from each other and from the work in flight. A collision that survives to the merge is a conflict even when the two worktrees never touched. The other fix is to have the exploration notes pin the exact names each ticket adds. This is the same advice as its sibling skill, and it is the reason the exploration pass is worth taking.

**A teammate died mid-ticket. What happens to its task?**

Ownership is never released automatically, so the task stays in progress under the dead name. The two failures look different, and the difference decides what to do: a creation that failed shows `failed` with the error in `diagnostics` and never claimed anything, while a member that died after claiming shows `inactive` with no report. Either way the recovery is a replacement under a new name, with the task reassigned to it, because names are never reused. An interrupted member is a different case and keeps its history, which is why it is the cheapest way to fix its own work.

**Cleanup failed and left every worktree behind.**

A compound `git worktree remove … && git branch -d …` dies as a whole when the second half is refused by a branch guard, so the first half never runs either. Run each removal as its own command. The worktree removal is forced, which is what a checkout with leftover build output needs. A branch that `-d` refuses after integration history was rewritten (cherry-picked for a linear pull request, or amended after review) is the user's call, not the skill's.

## It's working if

- Every teammate you need is visible in one list, and you can say which ticket each one holds.
- You steer a teammate that went wrong instead of respawning it, and its history comes with it.
- The number of members you created is the number you budgeted at the start, not one more per ticket.
- The frontier comes from the board, so a ticket unblocks the moment its merge lands rather than at the end of the run.
- A worktree appears when its ticket joins the frontier, cut from the branch tip of that moment, and never sits waiting on a blocker.
- No implementer started before the exploration notes existed.
- Every worker wrote into its own worktree.
- Every merge happened in the Lead's checkout, and every board task closed after it.

## Where it fits

The Agent Teams half of the implementation flow: the pick when your harness has durable teammates, with [implement-spec](https://aihero.dev/skills-implement-spec) as its plain-sub-agent sibling. It closes out on the integration branch with [code-review-agentteam](https://aihero.dev/skills-code-review-agentteam), which runs the two review axes as teammates.

[ask-matt](https://aihero.dev/skills-ask-matt) routes over the whole set.
