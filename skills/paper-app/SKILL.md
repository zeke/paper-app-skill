---
name: paper-app
description: Work in the Paper design app (paper.design) through its local MCP server (the mcp__paper tools). Use when asked to design, mock up, or explore variants in Paper, edit artboards or slides on a Paper canvas, take screenshots or export PNGs, PDFs, or videos from Paper, move or clone Paper files, use custom fonts or local images in Paper, or turn a Paper design into code. Covers codemode quirks, result parsing, export and screenshot workarounds, canvas organization, and what the MCP can't do.
---

# Paper app

[Paper](https://paper.design) is a design tool whose canvas is real HTML and CSS. The Paper desktop app runs a local MCP server that lets agents read the canvas and write HTML into it as design nodes.

The server ships its own guide. This skill covers what that guide leaves out: things learned the hard way across real sessions.

## Before you start

1. Check that the tools exist: `await describeNamespace("mcp__paper")` in codemode. If the namespace is empty or `ALL_TOOLS` has no `mcp__paper__*` entries, the session didn't connect, usually because Paper wasn't open when the session started. The server lives at `http://127.0.0.1:29979/mcp` and only answers while Paper is open. `pi mcp list` (run from bash) shows whether the server itself is healthy. If it is, the session can't reconnect on its own: ask the user to run `/mcp reconnect paper` in pi, or `/reload` if that doesn't work.
2. Load the official guide once per session: `get_guide({ topic: "paper-mcp-instructions" })`. Follow it: post a design brief before new designs, write one visual group per `write_html` call, review with screenshots, and call `finish_working_on_nodes` when done. Other topics: `mobile-status-bar`, `figma-import`, `image-generation`.
3. Find the file. `list_resources({ limit: 200 })` lists files without one being open. Pass `fileId` to every file-scoped call. `get_basic_info({ fileId })` works on a file that isn't the foreground tab, so you rarely need `open_file`, which opens a tab in the user's app.
4. If the project has an `AGENTS.md`, look for the Paper file ID and canvas conventions there before exploring.

## Calling Paper from codemode

- Run dependent Paper calls in one script with `await`, or in sequential turns. Never issue them as parallel tool calls in the same turn. Parallel calls caused most of the failures in past sessions, and each was misread as a Paper bug: "Node not found" for a node a sibling call had just created, children landing in scrambled order, exports missing recent writes, and `read` failing with ENOENT on an export that hadn't been written yet.
- Every file-scoped tool requires `fileId` in its codemode declaration, even though the server's instructions say you can omit it. Leaving it out fails validation. Only `get_basic_info` and `get_selection` accept no `fileId`.
- Run `describeTool("mcp__paper__<name>")` before the first use of a tool. Argument shapes aren't guessable: `get_font_family_info` takes `familyNames: string[]`, `set_text_content` and `update_styles` take an `updates` array, and `export` takes a `nodes` map.
- File-scoped tools return one plain string: a `{ file, contentHash }` header JSON block, then the body. Tools without a file (`list_resources`, `create_file`, `open_file`, `get_guide`) return `{ content: [{ type: "text", text }] }`. Don't regex the first `"id"` out of a result: it's the file ID, and passing it as a node ID fails with "Node not found".
- File-scoped failures and validation errors throw. Some non-file failures (like "Open a Paper file to use this tool.") resolve with `isError: true` instead. Don't wrap Paper calls in `.catch(() => {})`: a wrong argument shape then fails silently and the edit just never shows up.
- "Your Paper file is currently disconnected from the server" means writes fail while reads still work. Sleeping and retrying didn't fix it in past sessions. Ask the user to bring the Paper file back to the foreground, then retry.
- `get_screenshot` returns no image in codemode: its result is flattened to the header string. Use the `shot()` helper below, which exports a PNG and reads it in the same script. When a harness calls MCP tools directly instead of through codemode, `get_screenshot` images do come through.

Useful result shapes, after parsing the body:

| Tool               | Body                                                                                     |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `create_artboard`  | `{ id, name, worldX, worldY, width, height }`                                            |
| `write_html`       | `{ createdNodes: [{ id, name, component }] }`, top-level nodes only                      |
| `get_tree_summary` | `{ summary }`, an indented text tree with `Type "name" (id) W×H`                         |
| `get_children`     | `{ children: [{ id, name, component, childCount, x, y }], count }`                       |
| `find_nodes`       | `{ nodes: [{ id, name, component, matched: [{ styleName, styleValue }] }] }`             |
| `duplicate_nodes`  | `{ duplicatedNodes: [{ sourceId, newId, descendantIdMap }] }`                            |
| `export`           | `{ exports: [{ nodeId, filePath }] }`                                                    |
| `update_styles`    | Lists `ignoredStyles` when a style is inert in that context. Check it.                   |

### Helpers

Paste these at the top of a codemode script, or store them once and `eval` them in later calls:

```js
var F = "<fileId>";
var txt = r => typeof r === "string" ? r : (r?.content ?? []).map(c => c.text ?? "").join("\n");
// Strip the { file, contentHash } header that file-scoped tools prepend.
var body = r => { const t = txt(r); const i = t.indexOf("\n}\n"); return t.startsWith('{\n  "file"') && i > -1 ? t.slice(i + 3) : t; };
var json = r => JSON.parse(body(r));
var w = async (target, html, mode = "insert-children") =>
  json(await tools.mcp__paper__write_html({ fileId: F, targetNodeId: target, mode, html })).createdNodes;
// Export a node to PNG, show it, then delete the file from ~/Downloads.
var shot = async (id, scale = "1x") => {
  const { exports } = json(await tools.mcp__paper__export({ fileId: F, nodes: { [id]: [{ format: "png", scale }] } }));
  const p = exports[0].filePath;
  image(await tools.read({ path: p, offset: null, limit: null }));
  await tools.bash({ command: `rm -f "${p}"`, timeout: 5 });
};
```

To reuse helpers across calls: `store("paper", helperSource)`, then `eval(load("paper"))` at the top of each later script. Declare helpers with `var`, because `const` and `let` inside `eval` don't leak out. `store()` only keeps values when the script succeeds, so store helpers in a script that does nothing else.

## Screenshots and exports

- Take screenshots in a separate codemode call from the edits. An export in the same script as the writes can run before Paper renders them, so the PNG shows missing or stale content.
- `export` writes to `~/Downloads/<artboard name>.png`, with `@2x` or `@0.5x` for other scales and ` (1)`, ` (2)` suffixes when a file with that name exists. Characters like `?` and `:` become `_`. Always use the returned `filePath`. Don't pick files with `ls -t ~/Downloads` or name globs: that has picked up stale exports from earlier runs.
- Move or delete exports after reading them so they don't pile up in `~/Downloads`. Leave files you didn't create alone.
- Use `0.5x` for review screenshots to save tokens, `2x` for deliverables (1920×1080 artboards become 3840×2160).
- `export_combined_pdf` merges artboards into one PDF, ordered by canvas position. `export` also does `mp4`/`webm` video, plus `svg` for SVG nodes, `jpg`, `webp`, `avif`, and `pdf`.
- For turning artboards into Google Slides, see [references/slides-export.md](references/slides-export.md).

## Writing HTML

The `write_html` tool description lists the rules (inline styles, flex, no `margin`, no `display: grid`, no tables, `layer-name` for layer names). Additional findings:

- An empty `<div>` becomes a Rectangle node, even with `display: flex`, and Rectangles can't have children. Writing into one fails with "cannot have children". Write a container together with its first child, or replace the Rectangle with `write_html` in `replace` mode.
- Rich text isn't supported. For syntax-highlighted code or a sentence with one colored word, build a flex row of separate text nodes, one per token, each with `white-space: pre`. Code becomes one row per line. Generate the HTML with a helper rather than writing it by hand.
- A `background` with gradient layers ignores `backgroundColor`. Add the base color as the last layer: `linear-gradient(#F6F7F9, #F6F7F9)`.
- Inline `<svg>` works for icons, logos, and connector lines, and takes `fill`/`stroke` colors and token variables. Inline a brand logo's SVG instead of linking to it.
- SVG path fills are invisible to `find_nodes` and `get_computed_styles` (which returns `{}` for them). Find the `SVGVisualElement "Path"` nodes in `get_tree_summary` and recolor them with `update_styles({ styles: { fill } })`, which does work.
- Local images need an absolute path with the `paper-asset://` scheme: `<img src="paper-asset:///private/tmp/spec/foo.png">`.
- Equal-width columns in a flex row need `flex: 1; flex-basis: 0`. Padding on one column otherwise makes it wider than its siblings.
- If content clips, set the artboard to `height: "fit-content"` with `update_styles` instead of guessing a new height.

## Fonts

- `get_font_family_info({ fileId, familyNames: [...] })` checks fonts installed on the machine and Google Fonts. Call it before the first typographic styling in a session.
- Fonts installed mid-session (e.g. into `~/Library/Fonts` on macOS) don't show up until Paper restarts. Ask the user to quit and reopen Paper. Don't kill it yourself, since that can lose their state.
- Check a font's license before it goes on a public site. Many dafont.com fonts are free for personal use only.
- To compare many fonts without restarting Paper, render specimens to transparent PNGs locally (e.g. with Pillow), place them as `paper-asset://` images on a specimen artboard, then install only the finalists.

## Organizing the canvas

- When iterating on a design, duplicate the artboard and change the copy. Don't edit earlier boards in place: users want to see the progression and often ask to go back to an earlier version. One user had to ask for this twice in a session.
- `create_artboard` and `duplicate_nodes` drop new artboards into the best empty spot, which scatters them. To arrange them, set `top` and `left` on the artboards with `update_styles`. Keep at least 80px between artboards.
- Keep design explorations apart from the chosen direction: in their own region of the canvas, or on their own page (`create_page`, then pass `pageId`). You can't switch which page the user is viewing.
- For decks and variant sets, use a fixed grid (e.g. four per row, `x = x0 + col × (width + gap)`). When an artboard is added or removed, reflow the rest so there are no gaps, and renumber their names.
- For light and dark variants, put the light row on top and the matching dark version directly below each one. To make the dark versions, see [references/dark-mode-variants.md](references/dark-mode-variants.md).
- Give artboards descriptive names like `Homepage · A (Editorial)` or `04 · What are Workers?`. Export filenames come from them.
- Node IDs don't outlive the session. Write the file ID, URL, naming scheme, grid formula, and style rules into the project's `AGENTS.md` so the next session can pick up.

## Design to code

- Read exact values with `get_jsx` (`format: "inline-styles"` or `"tailwind"`) and `get_computed_styles`. Never measure sizes or colors from a screenshot.
- Computed colors can come back as `oklab(...)`. Copy them verbatim instead of converting them.
- After building the page, compare it in a real browser against a Paper export, in light and dark modes.

## What the MCP can't do

- Share files or manage permissions. The user shares from the Paper app: invite people to the team to let them edit, or send the file URL for view-only access.
- Move files into folders. New files from `create_file` land outside the user's folders, and the user has to drag them in.
- Move nodes between files. `move_nodes` only moves within a file, across pages. To move a whole file's contents, `create_file({ cloneFileId, name })`, check that node and artboard counts match, then `delete_nodes` the originals. Confirm with the user before deleting.
- Switch the page the user is looking at.
