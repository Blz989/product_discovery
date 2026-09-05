# Folder storage prototype

`folder-storage.html` tests whether storing each idea as its own file, with images as real binaries,
holds up better than one JSON document once rich text and screenshots are in play.

Open it in Chrome or Edge, click **Choose folder**, and point it at an empty directory.

## What it writes

```
<your folder>/
  project.json              name, key sequence
  ideas/
    IDEA-1.json             fields plus the description as HTML
    IDEA-2.json
  assets/
    IDEA-1/
      k3f9a2xq.webp         a real image file, never base64
```

Descriptions reference images as `<img data-asset="k3f9a2xq.webp">`. Blob URLs are attached when the
idea is displayed and stripped before writing, so no transient URL is ever persisted.

## Try this

1. **Add sample ideas**, then open one.
2. Paste a screenshot straight into the description, or use **Image**. Watch the log: it reports the
   original size, the stored size after downscaling to 1600px and re-encoding to WebP, and the
   saving.
3. Add several more images across a few ideas.
4. Click **Compare with single file**. It builds the equivalent monolithic document, base64 encoding
   every asset, and times serializing and writing it once. That is what the current app does on
   every save.

## What it showed here

Measured with one synthetic 2400x1400 screenshot across four ideas:

| | Folder | Single file |
| --- | --- | --- |
| Total size | 262 KB | 349 KB (1.33x) |
| Written per edit | 328 B, one file | the whole document |
| Time to write | 0.3 ms | 28 ms |

The size gap is the base64 tax and grows linearly with images. The time gap matters more: the
single-file cost is paid on **every keystroke** and scales with the whole corpus, while the folder
cost stays flat at one small file no matter how large the project gets. Ninety-two times here, and
that ratio widens with every screenshot added.

## Also proven

- **Migration.** **Import project JSON** takes an existing single-file project and explodes it into
  one file per idea, converting plain-text descriptions to HTML paragraphs.
- **Portability.** **Export single JSON** collapses the text back into one document, so nothing is
  trapped by the folder layout.
- **Orphan cleanup.** **Clean orphan assets** removes images no description references, and asset
  folders whose idea is gone. Assets are always written before the JSON that references them, so an
  interruption leaves a harmless orphan rather than a broken reference.
- **Sanitising.** Pasted HTML is filtered to an allowlist. Scripts, event handlers and
  `javascript:` URLs are stripped.

## Limits worth knowing

- Chrome and Edge only. It uses `showDirectoryPicker`, which Safari and Firefox do not implement.
- The folder handle is remembered in IndexedDB, but the browser may ask you to re-grant permission
  when you come back.
- Startup reads every idea file. Fine for hundreds of small files read in parallel, and the log
  prints the timing so you can watch it. If it ever drags, an index file is the fix, but do not
  build one before the numbers ask for it.
- Config here is a single `project.json`. For multiple editors it would need splitting per entity;
  see `docs/handoff/2026-09-04-multi-user-rearchitecture.md`.
- This is a lab, not the app. It shares the storage design, not the code.

## Status

Exercised end to end against an in-memory filesystem: seeding, per-file writes, image storage and
downscaling, rehydration after reload, the sanitiser, statistics, the single-file comparison, import
and orphan cleanup. Not yet run against a real directory picker, which needs a human click.
