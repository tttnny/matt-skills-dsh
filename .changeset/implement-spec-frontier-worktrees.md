---
"mattpocock-skills": patch
---

`implement-spec` and `implement-spec-agentteam` staff the frontier instead of opening every worktree up front: a worktree is created when its ticket becomes ready, cut from the integration branch tip of that moment, so a ticket that starts after its blockers merged no longer begins on a stale base. Finished branches go to the merger the moment they report, which is what unblocks the next tickets, and the new frontier is staffed from the merged tip.
