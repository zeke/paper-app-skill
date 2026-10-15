# Dark mode variants of existing artboards

A recipe for making dark versions of finished light artboards by remapping colors, instead of rebuilding them. It came from a session that made dark versions of two dashboard mockups.

## Steps

1. Take the dark palette from the project's code if it has one (CSS variables, `prefers-color-scheme` blocks), so the mockups preview what the real page will look like. Don't invent colors when real ones exist.
2. `duplicate_nodes` the light artboards. Keep the `newId` values with `store()` so later scripts can find them. Move each copy next to its light version with `update_styles` on `top` and `left`, because duplicates land wherever Paper finds empty space.
3. For each light color, run `find_nodes({ fileId, nodeId: darkArtboardId, filters: [{ styleValue: "#FFFFFF" }] })`. Each result's `matched` array names the property that matched (`backgroundColor`, `color`, `borderTopColor`, and so on), so one light color can map to different dark colors depending on the property. A near-black text color usually becomes a light text color, but a near-black panel becomes a slightly lighter dark panel.
4. Build one `update_styles` batch per artboard: `{ nodeIds: [id], styles: { [styleName]: darkValue } }` for each match.
5. Export both artboards and look at them before calling it done.

## Pitfalls

- Mapping by value alone catches things you didn't mean. Mapping `#FFFFFF` to the dark background also turned white text on accent-colored buttons and cards dark. Fix it by searching for the new dark value and restoring every match whose `styleName` isn't a background.
- SVG path fills don't show up in `find_nodes` or `get_computed_styles`. Find `SVGVisualElement "Path"` nodes in `get_tree_summary` and set `fill` on them directly.
- A straight swap isn't always right. Elements that stood out by being dark on a light page need a different treatment in dark mode, like a lighter panel with a hairline border. Accent colors often need brightening to read on near-black. Shadows mostly disappear.
