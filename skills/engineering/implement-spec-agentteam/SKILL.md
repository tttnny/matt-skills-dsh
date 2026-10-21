---
name: implement-spec-agentteam
description: "Implement the result of /to-spec and /to-tickets in code with DSH Agent Teams."
disable-model-invocation: true
---

You have been provided a spec. This spec should have tickets associated with it, describing how to implement the spec.

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`.

The goal is the entire spec implemented on a single **integration branch**, with every ticket resolved the way the issue tracker closes work.

The tickets are not a list of steps. They are a **task graph** with blocking relationships between them. This means there is always a **frontier** of tickets which are ready to be grabbed.

This is the **Agent Teams** form. Every worker is a durable **teammate**: a member you can list, wait on, wake with a message, and interrupt. Ticket state lives on the shared **task board**, which every member reads. Communicate through **context pointers** (the spec, the ticket, the board, the shared notes, earlier commits) rather than pasted summaries.

Reach for this form whenever the Agent Teams tools are available. Every worker being listable is the point: you can see who is running, wait on one member without blocking the rest, and interrupt a member that has gone sideways.

## Steps

### 1. Read the spec and the tickets

Read the spec and every ticket, and write down the graph: which ticket blocks which.

Done when you can name every ready ticket and its blockers without going back to the tracker.

### 2. (optional) Explore first, and wait for it

If the tickets need facts you don't have (codebase files, external docs), spawn one **exploration teammate** before anyone builds anything. Tell it to save its notes as markdown in a directory outside the repo, so every later teammate can read them.

**No implementer starts while it is still working.** Its notes are what those implementers read, and a teammate that starts first cannot read notes that do not exist yet. Wait for its report, then carry the note path in every later task as a context pointer.

Done when the notes exist and you have read them.

### 3. Create the integration branch

Create the branch the whole run lands on. If the issue tracker closes work through PRs, or the user asks for one, open a draft PR after the first merge in step 7 (a branch with no commits ahead of main can't open one), marked as closing the spec and tickets.

Done when the branch exists and the tickets' base commit is on it.

### 4. Put the graph on the shared task board

The board is the single source of truth for what can start; the tracker is only where the graph came from.

Create one task per ticket with `team_task_create`, `blocked_by` naming the tickets that must land first. An implementer's ticket is not done when it reports, it is done when its work is merged, so the board task is completed by you at the merge (step 7) with `team_task_update`, not by the implementer: the merger cannot close it, because only a task's owner or the Team Lead may transition one.

Done when every ticket has a board task and the `blocked_by` edges match the graph you wrote in step 1.

### 5. Create the dedicated merger

Create the **dedicated merger teammate** now, once, and keep it for the whole run. Every merge in step 7 goes to this one teammate: same standards, same context, one member.

Done when the merger exists and you have told it how you will hand it work.

### 6. Staff the frontier, one teammate per ticket

For each ready ticket, and only those, read the integration branch tip as it stands right now, create the worktree and branch from that commit, then spawn a teammate into it. A teammate that has to build its own workspace starts by paying for setup it was not asked to do, and one that starts on the wrong base lands work in the wrong place.

A blocked ticket holds no worktree yet. One cut before its blockers land is already behind by the time it starts, and turns a merge that should be a fast-forward into a conflict resolution.

Spawn one teammate per ready ticket with `spawn_teammate`. Each gets a complete, self-contained task: it starts with no history, so everything it needs must be in the prompt.

Each implementer's prompt carries:

- its board task id, and which ticket it came from (a context pointer, not a copy);
- an instruction to confirm its worktree is based on the integration branch tip it was handed, and to reset onto it if not;
- the worktree path and branch to work in, the integration branch to base on, and the tip commit its worktree was cut from;
- an instruction to call the Skill tool with `tdd` and build the ticket red-green;
- an instruction to merge the integration branch tip into its own branch before reporting done, so landing it is a fast-forward;
- an instruction to report back to `lead` with: branch, commit, files changed, and any acceptance criterion it did not meet.

Done when every ready ticket has a worktree, a teammate working in it, and each one holds the pointers above.

### 7. Merge each finished ticket with the dedicated merger

A teammate reporting done is not a landed ticket. The moment one reports, hand the finished branch to the **dedicated merger** with `send_message`, ahead of whatever else you were about to do, and let it merge into the integration branch and resolve conflicts. A report left waiting holds back every ticket blocked behind it and widens the gap between that worktree's base and the integration branch.

Then complete that ticket's board task. Only the merged state counts.

If a merge reveals a real conflict of intent rather than a textual one, interrupt the implementer that owns it, send it the conflict, and let it resolve its own ticket. An interrupted member keeps its history, so it is the cheapest way to fix its own work.

Done when the ticket's branch is on the integration branch and its board task is complete.

### 8. Re-read the board and staff the new frontier

Merging a ticket unblocks others. Re-read the board and the integration branch tip as the merge left it, then spawn a teammate for each newly ready ticket, exactly as in step 6, so each new worktree is cut from the merged tip rather than the one the run started on.

Done when no ready ticket is unstaffed and no blocked ticket is staffed.

### 9. Review the integration branch

Once every ticket is on the integration branch, call the Skill tool with `code-review-agentteam` on it. That runs the two review axes as teammates.

Send every finding to a single implementer teammate to fix, then re-check the findings that were raised. Expect the first review to find real problems: the run's output is a draft that the review completes, not something to ship on its own.

Done when every raised finding is either fixed or answered.

### 10. Close out

If a draft PR exists, mark it ready for review. Otherwise, resolve each ticket the way the issue tracker closes work, and report the integration branch.

Then remove every implementer worktree you created, and leave the integration branch standing.

Done when no worktree from this run is left behind and every ticket is resolved.

## Working the team

- **Waiting on a member.** Check `list_agents` first: `wait_agent` observes changes that happen after it is called, and returns immediately with nothing when no member is running. If the member you need is inactive, `send_message` wakes it, then wait.
- **Inactive is not done.** It only means no turn is executing. Read the board and the member's report for what actually happened.
- **A stale member.** `interrupt_agent` it; its inbox is preserved. Send it the correction with `send_message` and let it continue. Spawning a replacement to do the same work loses the work in flight.
- **A blocked ticket.** Never staff one, even when a member is idle.
