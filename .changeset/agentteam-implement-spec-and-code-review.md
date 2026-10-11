---
"mattpocock-skills": minor
---

Added `implement-spec-agentteam` and `code-review-agentteam`, the DSH Agent Teams forms of the spec implementation and review flows. In the Agent Teams form every worker is a durable teammate: listable, waitable, wakeable with a message, and interruptible, so a stuck or wrong-headed worker can be steered instead of abandoned. The ticket graph lives on the shared task board as the single source of truth for what can start, one dedicated merger teammate handles every merge for the whole run, and the optional exploration pass gates the whole run: no implementer starts before its notes exist. Both skills are language- and repo-neutral, so nothing in them names a specific verification command or directory layout.
