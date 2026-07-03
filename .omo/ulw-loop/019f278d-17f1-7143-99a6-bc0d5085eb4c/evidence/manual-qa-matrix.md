# Manual QA Matrix

Task: supplement chapter 5 consistent hashing draft from photographed references.
Surface: local Markdown artifact and CLI-readable evidence files.

| Criterion | Surface | Invocation | Expected | Result | Artifact |
| --- | --- | --- | --- | --- | --- |
| C001 summary coverage | CLI/filesystem | `rg -n '설계 관점 정리|참고자료별 설계 포인트|가상 노드|Dynamo|Cassandra|Maglev|Discord' seonghun/chapter05-consistent-hash/Untitled.md` | Korean summary covers core ring, virtual nodes, Dynamo, Cassandra, Maglev, Discord, and references | PASS | `C001-rg-summary.txt` |
| C002 tradeoff coverage | CLI/filesystem | `rg -n '한계|트레이드오프|불균등|가상 노드|hot path|lookup|성능' seonghun/chapter05-consistent-hash/Untitled.md` | Notes mention non-uniform distribution, virtual-node/load movement, and lookup/performance cost | PASS | `C002-tradeoffs.txt` |
| C003 preservation | CLI/filesystem | `sed -n '1,24p' seonghun/chapter05-consistent-hash/Untitled.md` | Original rough draft remains before the new reference section | PASS | `C003-preservation.txt` |

Cleanup receipt: no server, tmux, browser, container, bound port, or temp runtime state was created. Obsolete zero-byte `C003-diff.txt` was removed.
