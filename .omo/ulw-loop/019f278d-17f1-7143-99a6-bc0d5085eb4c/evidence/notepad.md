# ULW Notepad

Skill survey:
- Used `omo:ulw-loop` because the user explicitly requested `$omo:ulw-loop`.
- Did not use `omo:ulw-research`; the research scope was bounded to the photographed reference list rather than open-ended saturation research.
- Did not use `omo:programming`; no `.py`, `.ts`, `.go`, `.rs`, or source-code file was edited.
- Did not use `omo:remove-ai-slops`; no code generation or branch code cleanup was requested, and the artifact is Korean study Markdown.
- Did not use `omo:git-master`; the user did not request a commit, and the target draft is currently untracked.

Tier: LIGHT.
Shape: research-writing.
Reason: one Markdown artifact, no source code, no runtime app, no external integration, no DB, no concurrency, no security/session behavior.

Source notes:
- Wikipedia consistent hashing: basic definition and average remapping framing.
- Tom White: core ring model, same hash function for objects/caches, virtual nodes.
- Dynamo paper: consistent hashing for partitioning, virtual nodes/tokens, preference list and distinct physical nodes.
- Cassandra paper: coordinator on ring, replication, non-uniform distribution and heterogeneity tradeoff.
- Discord blog: ring lookup in hot path and runtime lookup optimization.
- Stanford CS168: algorithmic background and web-cache/distributed-storage motivation.
- Maglev paper: load-balancer use, connection tracking, lookup table, load balancing vs minimal disruption.

Self-review:
- Target file preserves the original rough draft at the top.
- Added section is Korean, reference-based, and framed as chapter-note material.
- Evidence artifacts are non-empty and map to the revised criteria.
- Dirty worktree contains pre-existing unrelated paths (`.DS_Store`, `seonghun/.DS_Store`, `.gitignore`, `seonghun/trash/`) plus this ULW session and target Markdown. These unrelated paths were not modified for this task.
