# AGENTS.md

Technical instructions for agents working in this repo.

Keep this file current. Whenever the skill's structure, install instructions, or conventions change, update this file in the same change.

## What this repo is

A single agent skill, `paper-app`, for working in the [Paper](https://paper.design) design app through its local MCP server (`mcp__paper` tools). It follows the [agentskills.io](https://agentskills.io/specification) spec and installs with `npx skills add zeke/paper-app-skill`.

## Structure

```
skills/paper-app/SKILL.md                    skill entry point (frontmatter: name, description)
skills/paper-app/references/slides-export.md Paper artboards to Google Slides recipe
skills/paper-app/references/dark-mode-variants.md  dark versions of light artboards by color remapping
```

Skills live at `skills/<name>/SKILL.md`, the layout the `npx skills` CLI discovers. The directory name must match the `name` frontmatter field.

## Conventions

- `SKILL.md` is always-loaded context. Keep it concise and push long recipes into `references/`.
- Don't restate the server's own guide (`get_guide({ topic: "paper-mcp-instructions" })`) or tool descriptions. The skill covers what they leave out.
- Every claim should come from a real session or a live check against the MCP. Mark anything unverified as such. Paper ships often, so re-check quirks (result shapes, `get_screenshot` in codemode, export paths) against the live server when editing them.
- No bold in prose, no em dashes, no headings smaller than H2.
- No build, lint, or test scripts are configured.

## Sources

The findings came from pi sessions in Oct 2026: a slide deck (`~/git/zeke/cf-bindings-research`), a homepage and wordmark exploration (`~/git/zeke/slop.how`), CTA variants and a file move for cfdev.school, and a Paper-to-Google-Slides export (`~/git/zeke/cloudflare-connect-2026`). `~/git/zeke/cf-bindings-research/AGENTS.md` has an earlier "Paper MCP gotchas" list.

A second pass mined the pi sessions under `~/.pi/agent/sessions/` that made real Paper calls: an LLM context window visualization (`llm-context-window-visualization`, direct `mcp` tool calls), the aly.wtf layout explorations (`writing-skills`), and the cfdev.school dashboard and dark mode variants (`cloudflare-dev-school`). Many sessions mention `mcp__paper` only because the tool list is in the system prompt, so to find new ones, parse the JSONL and count `toolCall` entries whose name or arguments reference `mcp__paper__` or `paper_`. When reviewing a session, check whether a failure came from parallel tool calls before recording it as a Paper quirk.

## Publishing

- Install with `npx skills add zeke/paper-app-skill`. No build step.
- Don't push directly to `main` unless asked.
