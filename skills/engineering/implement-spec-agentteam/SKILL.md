---
name: implement-spec-agentteam
description: "Implement the result of /to-spec and /to-tickets in code with DSH Agent Teams: the ticket graph is a shared task board, and each ticket is built by a named subagent the Lead can message and interrupt."
disable-model-invocation: true
---

You have been provided a spec. This spec should have tickets associated with it, describing how to implement the spec.

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`.

The goal is the entire spec implemented on a single **integration branch**, with every ticket resolved the way the issue tracker closes it.

The tickets are not a list of steps. They are a **task graph** with blocking relationships between them, so there is always a **frontier** of tickets which are ready to be grabbed.

This is the **Agent Teams** form. The graph is copied onto the shared **task board**, which is where every later step reads the frontier from, and every worker is a named **subagent** of the Lead: listable, addressable by name, steerable with a message, and stoppable with an interrupt. Talk to them through **context pointers** (the spec, the ticket, the board, the notes, earlier commits) rather than pasted summaries.

If `team_task_create` is missing from your tool list, Agent Teams is off for this session. Say so and run `/implement-spec` instead: the same graph, worked without a board.

## Steps

### 1. Read the spec and the tickets

Read the spec and every ticket, and write down the graph: which ticket blocks which.

Done when you can name every ready ticket and its blockers without going back to the tracker.

### 2. Create the integration branch

Create the branch the whole run lands on and stay in that checkout, which is the only one you commit in. If the tracker closes work through PRs, or the user asks for one, open a draft PR after the first merge (a branch with no commits ahead of main cannot open one), marked as closing the spec and its tickets.

Done when the branch exists, the tickets' base commit is on it, and your checkout is clean.

### 3. (optional) Explore first, and wait for it

If the tickets need facts you do not have (codebase files, external docs), spawn one **exploration subagent** before anyone builds anything. Point it at an absolute notes path inside the integration checkout, under a path that checkout ignores (for example under `.scratch/`), and tell it to report that absolute path.

**No implementer starts while it is still working.** Its notes are what the implementers read. Wait for its completion notice, read the notes, then carry the path in every brief as a context pointer.

Done when the notes exist and you have read them.

### 4. Copy the graph onto the board

Create one task per ticket with `team_task_create`, walking the graph in dependency order so a blocker always exists before it is named. `blocked_by` takes **board task ids**, not tracker ids, and `write_scopes` takes the workspace-relative paths that ticket expects to write, so two independent-looking tickets that share a file are visible as a collision before either starts.

Keep a mapping from ticket to board task id; every later step uses the board id.

Done when `team_task_list` shows one task per ticket, every blocker resolves to a real task, and the ready tasks are exactly the frontier you wrote down in step 1.

### 5. Staff the frontier

Read `team_task_list({ ready: true })`. For each ready task:

- cut its worktree and branch from the integration branch tip as it stands now (`git worktree add <path> -b <branch> <integration-branch>`), with the path outside the checkout or under one the checkout ignores;
- spawn one implementer as a **named subagent**: `subagent({ name, description, cwd: <worktree>, prompt })`.

A blocked ticket holds no worktree. A base cut before its blockers landed is already behind by the time it starts, and turns a merge that should be a fast-forward into a conflict.

The brief is self-contained, because the child sees none of this conversation. It carries:

- the ticket text and its acceptance criteria, or a pointer to the ticket;
- the branch name and the integration commit it was cut from;
- the board task id, and the instruction to claim it with `team_task_get` then `team_task_update({ action: 'claim' })` before writing code;
- the paths it owns, and the instruction to stay inside its worktree;
- the absolute notes path from step 3, when there is one;
- the instruction to drive `tdd`, then report its branch name and test output.

Read the ready tasks' write scopes before staffing and start a batch whose scopes are disjoint. The board only warns about overlaps with tasks already in progress, so it is silent about the batch you are about to start.

Issue every spawn in one message and then stop working. Each completion notice starts a new turn for you.

Done when every ready task has a running implementer and you have no work left that is not a merge.

### 6. Merge on report, then close the task

The moment an implementer reports, merge its branch into the integration branch in your checkout, ahead of anything else you were about to do. A report left sitting holds back every ticket blocked behind it.

Then, in this order: run the repo's build and tests on the merged tip, and only if they pass complete the board task with `team_task_update({ action: 'complete', expected_revision })`. A ticket is done when its work is merged, not when its implementer says so.

Completing a task is what makes its dependents ready. Go back to step 5 with the new frontier.

Done when the board has no task left pending or in progress.

### 7. Close out

Call the Skill tool with `code-review-agentteam` on the integration branch, with the commit the branch was cut from as the fixed point. Fix every raised finding in **one** fresh implementer subagent, on its own worktree, and merge that branch the same way.

Then mark the draft PR ready if there is one, resolve each ticket the way the tracker closes work, remove every worktree and branch of this run, and report the integration branch.

Done when no worktree from this run is left behind and every ticket is resolved.

## Working the team

**Address members by the name you gave them**, and keep names short and stable: `ticket-<id>` reads well in `list_agents` and in the board's owner column.

**Steer before you replace.** A child that has gone the wrong way keeps its history: `interrupt_agent` stops its turn, and `send_message` redirects it with the correction. Respawning throws away everything it read and pays for it again, so it is the last resort.

**Do not poll.** `list_agents` is for naming who is running when you need to message someone, not for watching progress. Completion notices arrive on their own; `wait_agent` is for the case where the next thing you do needs their report inside this same turn.

**The board does not wake anybody.** A task change ends no wait and starts no idle member. When a member must act on a new task, send it a message.

**Replace a dead member on its branch, not from scratch.** A member that stalls or dies leaves its task in progress under its own name, because ownership is never released automatically. Reassign that task to a replacement with `team_task_update({ action: 'reassign', owner })` (the Lead is the only one who can), give the replacement a new name, and tell it to start from the dead member's branch so its commits are not rebuilt.

**You are the only writer to the integration checkout.** Implementers write only inside their worktree; you merge, you test the merged tip, and you close the task. Two members running Git in one checkout contend on one index that nothing guards.

**Delegation depth is host-configured.** Implementers do their own work. Do not tell one to spawn its own workers unless this session allows another level.
