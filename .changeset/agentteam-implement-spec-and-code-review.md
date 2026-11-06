---
"mattpocock-skills": minor
---

Added `implement-spec-agentteam` and `code-review-agentteam`, the same two flows with every worker a named subagent of the caller. An implementer or reviewer keeps its history, so a worker that goes the wrong way is redirected with `interrupt_agent` and a message instead of being abandoned and respawned, and a reviewer a finding came from can be asked about it without paying for the reading twice. Implementers are spawned with `cwd` set to their own worktree, so `read`, `edit` and `write` land there along with `bash`, and both flows spawn with `subagent` rather than `subagent_fork`, which would carry the caller's completed turns into a context that was meant to start clean. Nothing else about either flow changes: the same task graph, the same integration branch, the same two review axes and the same smell baseline.
