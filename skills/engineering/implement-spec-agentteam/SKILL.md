---
name: implement-spec-agentteam
description: "Implement the result of /to-spec and /to-tickets in code with DSH Agent Teams."
disable-model-invocation: true
---

You have been provided a spec. This spec should have tickets associated with it, describing how to implement the spec.

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`.

The goal is the entire spec implemented on a single **integration branch**, with every ticket resolved the way the issue tracker closes work.

The tickets are not a list of steps. They are a **task graph** with blocking relationships between them. This means there is always a **frontier** of tickets which are ready to be grabbed.

This is the **Agent Teams** form. Every worker is a durable **teammate**, so a ticket that stalls can be interrupted and steered instead of abandoned. Ticket state lives on the shared **task board**, which every member reads. Communicate through **context pointers** (the spec, the ticket, the board, the shared notes, earlier commits) rather than pasted summaries.

You are the only member who spawns and merges, and you close every ticket at the merge. Every teammate starts in your directory, so its checkout has to be handed to it.

## Steps

### 1. Read the spec and the tickets

Read the spec and every ticket, and write down the graph: which ticket blocks which.

Done when you can name every ready ticket and its blockers without going back to the tracker.

### 2. (optional) Explore first, and wait for it

If the tickets need facts you don't have (codebase files, external docs), spawn one **exploration teammate** with `context: "fresh"` before anyone builds anything. It spends one implementer slot, so count it in step 5. Tell it to save its notes as markdown in the integration checkout, under a path the checkout ignores, and to report that absolute path.

**No implementer starts while it is still working.** Its notes are what those implementers read. Wait for its report, then carry the note path in every later task as a context pointer.

Done when the notes exist and you have read them.

### 3. Create the integration branch

Create the branch the whole run lands on. If the issue tracker closes work through PRs, or the user asks for one, open a draft PR after the first merge in step 7 (a branch with no commits ahead of main can't open one), marked as closing the spec and tickets. When `gh` is not available or not authenticated, report the integration branch instead and let the tracker close the tickets.

Done when the branch exists and the tickets' base commit is on it.

### 4. Put the graph on the shared task board

The board is the single source of truth for what can start; the tracker is only where the graph came from.

Create one task per ticket with `team_task_create`, in dependency order, and give each one the `write_scopes` its ticket will touch. `blocked_by` takes the **board task ids** (`task-1`, `task-2`, ...) returned by those calls, never tracker ticket ids; an unknown id is rejected. The board warns only against tasks already in progress, so before staffing a batch, read the ready tasks' `writeScopes` and staff a set that is disjoint from each other and from the work in flight. A collision that survives to the merge is a conflict, even when the two worktrees never touch.

An implementer's ticket is not done when it reports, it is done when its work is merged. The implementer **claims** its task before starting and never completes it; you complete it at the merge. `complete` only accepts an in-progress task, so the claim is what makes the task closable.

Done when every ticket has a board task, every `blocked_by` edge matches the graph from step 1, and every task carries its write scopes.

### 5. Budget the team

The team cap counts every teammate the team has ever created, failed creations included, and no name is ever reused, so the pool is fixed for the whole run. Reserve two slots for the closing review and staff the frontier from what is left. The cap is a deployment value: a spawn that fails with `Team member limit <n> reached` reports it, which is the one way to learn it.

Reuse beats spawning. A finished implementer keeps its history, so `send_message` it the next ready ticket rather than creating a new member; spawn a new teammate only when every existing one is busy.

Done when the two review slots are reserved and every implementer you plan to staff fits inside the cap.

### 6. Staff the frontier

For each ready ticket, create its worktree and branch from the integration branch tip as it stands right now, then spawn a teammate and hand it that checkout. A teammate that has to build its own workspace starts by paying for setup it was not asked to do, and one that starts on the wrong base lands work in the wrong place.

You create the checkout: `git worktree add -b <ticket> <path> <integration>`. Keep the pool inside the session's workspace root, the directory the session started in, so the workspace-write root still covers the shared `.git`, and put its path in `.git/info/exclude` so the integration checkout stays clean for the closing review. `create_worktree` is not the tool here: it moves your own session into the new checkout, and you stay in the integration checkout.

Spawn one teammate per ready ticket with `spawn_teammate`, passing `context: "fresh"`. A fork inherits your completed turns, which is the opposite of what a ticket worker needs.

Each implementer's prompt carries:

- its board task id, and which ticket it came from (a context pointer, not a copy);
- the absolute path of its checkout, and `working_directory({ cd: <path> })` as its first action. That one call moves `read`, `edit`, `write`, and `bash` together, because all four resolve against the teammate's own directory; prefixing bash commands moves only bash;
- an instruction to claim its board task before it starts: `team_task_get` for the revision, then `team_task_update` action `claim` with that revision;
- an instruction to call the Skill tool with `tdd` and build the ticket red-green;
- an instruction to merge the integration branch tip into its own branch before reporting done, so landing is usually a fast-forward;
- an instruction to report with `send_message({ target: "lead", message: ... })` before its turn ends, carrying branch, commit, files changed, and any acceptance criterion it did not meet. A teammate that ends its turn without reporting leaves you nothing to act on.

Done when every ready ticket has a checkout, a teammate spawned with that checkout's path and the pointers above.

### 7. Merge each finished ticket yourself

A teammate reporting done is not a landed ticket. The moment one reports, merge its branch into the integration branch in your own checkout, run the build and tests there, then complete that ticket's board task. Only the merged state counts.

You are the merger. You complete the task at the merge, you own the final diff and the full test run, and a second member running Git in your checkout contends on one index that nothing guards.

Before completing, `team_task_get` the task and pass the revision it returns as `expected_revision`; every transition bumps the revision, so a value read before the spawn is already stale. If the integration branch moved after the implementer merged it in, merge it into the ticket branch once more before landing.

If a merge reveals a real conflict of intent rather than a textual one, interrupt the implementer that owns it, send it the conflict, and let it resolve its own ticket. An interrupted member keeps its history, so it is the cheapest way to fix its own work.

Done when the ticket's branch is on the integration branch, the build and tests pass there, and its board task is complete.

### 8. Re-read the board and staff the new frontier

Merging a ticket unblocks others. Re-read the board, and give each newly ready ticket to an idle implementer, or spawn one when every implementer is busy, exactly as in step 6.

Done when every ready ticket is staffed and every blocked ticket is unstaffed.

### 9. Review the integration branch

Once every ticket is on the integration branch, call the Skill tool with `code-review-agentteam` on it. That runs the two review axes as teammates, out of the two slots you reserved, with the commit the integration branch was cut from as the fixed point. It wants a clean tree: the integration checkout stays clean because you are the only member who commits there, the worktree pool is in `.git/info/exclude`, and the exploration notes live under an ignored path.

Send every finding to a single implementer teammate to fix, merge its branch when it reports, and re-check the findings that were raised. Expect the first review to find real problems: the run's output is a draft that the review completes.

Done when every raised finding is either fixed or answered.

### 10. Close out

If a draft PR exists, mark it ready for review. Otherwise, resolve each ticket the way the issue tracker closes work, and report the integration branch.

Then remove every checkout you created with `git worktree remove --force <path>`, delete its branch, and prune. Run each removal as its own command: a compound `&&` chain dies as a whole when the branch delete is refused. Leave the integration branch standing.

Done when no worktree from this run is left behind and every ticket is resolved.

## Working the team

- **Waiting inside `run_code`.** Keep each `wait_agent` well inside the enclosing deadline: `run_code` defaults to 120 seconds and caps at 600, so a wait longer than the deadline dies as `execution deadline reached` and reads as a tool failure. Wait in shorter hops and re-read the board between them.
- **A dead member.** Ownership is never released automatically. A member whose creation failed shows `failed` with the error in `diagnostics`; a member that died after claiming shows `inactive` with no report, and its task stays in progress under its name. Spawn a replacement under a new name, then `team_task_update` action `reassign` with `owner: <new name>` and the revision from `team_task_get`. `interrupt_agent` only stops a live turn, so it cannot recover a dead member.
