# Paper artboards to Google Slides

A recipe for turning a deck of Paper artboards into a Google Slides file, for when an event or reviewer requires Slides. It produces image slides: the text isn't editable in Slides, so edit in Paper and re-export.

## Steps

1. Make the artboards 1920×1080 (16:9, the Google Slides default) and name them in deck order, e.g. `01 · Title`, `02 · Agenda`.
2. Export every deck artboard as PNG at `2x` (3840×2160). Take the paths from the `filePath` values that `export` returns, then move the files into a working directory with clean, ordered names. Check each file's dimensions before using it.
3. Build a `.pptx` with python-pptx: a 13.333×7.5 in presentation, one blank-layout slide per PNG, each picture full-bleed. Set alt text on each picture through the `descr` attribute of its `cNvPr` element. Run it with `uv run` and an inline script dependency so nothing gets installed globally.
4. Upload the `.pptx` to Google Drive in the browser, open it, and use File > Save as Google Slides. Alt text survives the conversion. Delete the intermediate `.pptx` afterward.

## Why not the Slides API

`presentations.batchUpdate` with `createImage` needs a publicly reachable image URL, so the PNGs would have to be hosted somewhere first (a temporary tunnel works). It also needs Drive and Slides write scopes, which a read-only Google Workspace connection doesn't have. The `.pptx` upload avoids both.

## Alt text

Write alt text from the slide content. Text pulled from Paper with `get_node_info` or `get_jsx` loses structure on slides where code was built from per-token text nodes, so read the slide image or the source code instead.
