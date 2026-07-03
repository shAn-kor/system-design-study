# Manual QA Matrix

Task: replace abstract Mermaid diagrams with actual consistent-hash structure images in the chapter 5 draft.

| Criterion | Surface | Invocation | Expected | Result | Artifact |
| --- | --- | --- | --- | --- | --- |
| C001 | Markdown/filesystem | `rg -n '!\[.*\]\(images/' seonghun/chapter05-consistent-hash/Untitled.md` and `rg -n '```mermaid' ...` | Seven SVG image links and no Mermaid fences | PASS | `markdown-image-links.txt` |
| C002 | SVG render | `python3` XML parse + `rsvg-convert` to PNG | Seven valid SVGs and seven non-empty PNG renders | PASS | `svg-render-check.txt`, `rendered-png/*.png` |
| C003 | Markdown order | `rg -n '^## ' seonghun/chapter05-consistent-hash/Untitled.md` | Draft order preserved | PASS | `order-check.txt` |
| Visual spot check | Rendered PNG | `view_image` on `01-hash-ring-basic.png` and `07-vnode-add-delete.png` | Images show hash-ring and virtual-node structure pictures | PASS | rendered PNG files |

Cleanup receipt: no server, tmux session, browser, container, bound port, or long-running runtime process was created. First-pass reviewers were closed.
