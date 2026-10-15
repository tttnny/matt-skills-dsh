---
"mattpocock-skills": patch
---

`implement-spec` and `implement-spec-agentteam` claim each ticket as they take it, before its implementer or teammate starts, so a second run of the same spec skips the tickets already in flight instead of building them twice. Neither takes over a ticket another run has claimed, resolves only the tickets it took, and releases the claim on any it took but did not resolve. The GitHub, GitLab and local-markdown tracker templates gain a shared **Claiming a ticket** section that `/wayfinder` and both skills read; re-run `/setup-matt-pocock-skills` to refresh `docs/agents/issue-tracker.md`.
