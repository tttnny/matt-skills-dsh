---
"mattpocock-skills": patch
---

`code-review` and `implement-spec` now name the subagent mechanism they depend on. Workers are created with `subagent` rather than `subagent_fork`, so a child starts without the caller's completed turns; the caller issues every spawn in one message and lets each completion notice start the next turn; implementers are spawned with `cwd` set to their worktree, so `read`, `edit` and `write` land there along with `bash`; workers are named, so one that has gone the wrong way is redirected with `interrupt_agent` and a message instead of respawned; and reviewers are told to leave the tree alone, because a child holds `edit` and `write`.
