---
"mattpocock-skills": patch
---

`implement-spec` gives the dedicated merger subagent every finished branch to merge for the rest of the run, rather than a new merger per branch, and spells out why: one merger for the whole run holds merge standards consistent, and creating it before the first merge means no merge waits on a subagent being spawned.
