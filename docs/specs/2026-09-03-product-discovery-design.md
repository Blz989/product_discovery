# Product discovery tracker design

Date: 2026-09-03
Status: approved and implemented in `index.html`

## Goal

Replace a paid Jira Product Discovery seat for a single user with a tool that costs nothing to run and keeps its data in a file the user controls.

## Decisions

- **Single user, file-based.** One `.json` project file synced through Box or Drive. No backend, no accounts.
- **One HTML file.** No build toolchain, no dependencies beyond Google Fonts, which fall back to system fonts when offline.
- **Standalone.** Delivery work is a free-text ServiceNow demand reference on the idea (JSON key `delivery`), not a live integration.
- **Features in scope.** Ideas table with custom fields and a computed score, board by status, impact-vs-effort matrix, dated timeline roadmap plus Now / Next / Later buckets, insights and comments per idea, filters and search, Auto / Light / Dark theme.

## Approaches considered

1. Single HTML file using the File System Access API. Chosen. Works today in Chrome and Edge with zero setup. Safari and Firefox degrade to browser storage plus export/import.
2. Tauri or Electron desktop app. Works in any engine but adds a build chain and installers for a one-person tool.
3. Local Python server owning the file. Works everywhere but the user must start a process each session.

## Architecture

Everything lives in one file with four layers:

- **Data model.** `{version, name, seq, ideas[]}`. Each idea: `id, key, title, description, status, impact 1-5, effort 1-5, confidence 0-100, reach, labels[], owner, delivery, bucket, start, end, insights[], comments[], history[], created, updated`. `start` and `end` are `YYYY-MM-DD` strings and may be empty. Statuses and buckets are fixed lists. `fields[]` holds custom field definitions and `custom{}` on each idea holds their values.
- **Storage modes.** `mode` is `browser`, `file` or `folder`. Browser storage is always written as a
  cache. File mode serialises the whole document on a debounce, as before. Folder mode writes only
  what changed: `commit(change)` takes a hint (`{idea}`, `{ideas:[]}`, `{config:true}`,
  `{removed:key}`), which accumulates into a pending set and flushes 400 ms later, so a field edit
  costs one small file rather than the entire corpus. A `commit()` with no hint still marks
  everything dirty, which keeps the rarely-hit paths (import, manual reordering) correct at the cost
  of a full write. Deletes are explicit, since an absent idea cannot be inferred from a rewrite.
  Idea files are named by the idea's internal id rather than its key, and views and fields are one
  file per item, so no two concurrent writers can be routed to the same path by anything a user
  controls. Every write is compared against what was last written to that path and skipped when
  identical, which is what makes per-item config files cheap: a config flush touches only the item
  that actually changed. `project.json` deliberately carries no timestamp so it is not rewritten on
  every save and does not become a contention point; the display order of views and fields lives
  there, since order is a project-level fact and not a property of any one item.
- **Attribution.** `addHistory` stamps a `by` name onto each entry, and comments and insights carry
  the same field. The name lives in browser storage, not the project file: it identifies the person
  at this browser rather than a property of the project, and each person sharing a folder sets their
  own. It is omitted rather than stored empty when unset, so entries written before the setting
  existed, or by someone who left it blank, render with just a timestamp and no migration is needed.
  There is no authentication behind it, so it is an attribution convenience and not an access
  control.
- **Idea keys.** The key is a label, not an identity: files are named by the idea's id and links,
  merges and history all reference ids, so a key can be edited freely and a duplicate costs nothing
  but confusion. It is editable in the panel, normalized to upper case with a restricted character
  set, refused when another idea holds it, and recorded in history. `nextFreeKey(like)` returns the
  next unused number in the same prefix family and is deliberately pure, because it labels a button
  as well as minting a key and the two must agree; only `nextKey` advances `data.seq`. Duplicates
  are detected once per render into `dupSet` and surfaced in the Key column and the panel, with a
  one-click move to a free number. Sorting is by prefix, then by the number zero-padded, so mixed
  families group rather than interleave.
- **Folder refresh.** Folder mode polls the directory every 25 seconds, and on tab focus. The
  `lastModified` of every file read or written is recorded, so only files somebody else touched are
  parsed; an entity in the pending set is skipped so an unflushed local edit is never overwritten,
  and a poll is suppressed entirely while a write is in flight. Deletions are inferred from files
  that have disappeared. This is the only mechanism by which one tab learns about another; there is
  no conditional write in the File System Access API, so the design avoids collisions rather than
  detecting them, and simultaneous edits to one idea remain last-write-wins.
- **Folder migration.** Opening a folder written by an earlier version rewrites idea files from
  key-named to id-named and splits the single `views.json` and `fields.json` into per-item files,
  removing the originals, in one pass at adopt time.
- **Persistence.** Every change writes to `localStorage` immediately. When a file handle is connected, a debounced write goes to the file 700 ms later. The handle is stored in IndexedDB so the file reconnects after reload; the browser may require one click to re-grant permission. Export and import cover browsers without file access.
- **Views.** `render()` applies filters and sort, then dispatches to list, board, matrix, timeline, or bucket renderers. Board and buckets share one column renderer with HTML5 drag and drop.
- **Timeline.** The window is one fiscal year, chosen by `ui.tlFy` (empty means the current year and follows the calendar; a year number pins it; `all` restores the old auto range). Bars overlapping the window are clipped to it and marked, so a long-running idea shows where it crosses the boundary instead of being dropped or overflowing; ideas entirely outside are listed separately. There is one scale: the renderer measures the real scroll pane (inserted empty first, so its `clientWidth` already accounts for content padding and its own borders) and derives pixels per day so the window fits exactly with no horizontal scroll, with a floor that lets a very long "All dates" range scroll instead of becoming unreadable. A resize re-renders so the fitted scale stays correct. The today marker is positioned with `calc(var(--tl-side) + Npx)` rather than a JavaScript-measured offset, so it cannot drift from the label column when the layout reflows between rendering and printing; the phone layout is scoped to `screen` so paper never picks it up. Pointer events move a bar or resize either edge; the change commits as a history entry on release. A click without movement opens the idea.
- **Field registry.** Built-in fields (status, bucket, owner, labels, impact, effort, dates) and user-defined fields share one descriptor shape `{id, name, type, options[]}`. Status and bucket option lists live in `data.statuses` and `data.buckets` rather than in code, and `data.fieldNames` holds renamed built-in labels, so the field manager edits built-in and custom choice fields through one code path. Options are edited by id, not by name, so renaming one cannot orphan the ideas using it; deleting one reassigns its ideas first. Custom definitions live in `data.fields`; values live on each idea under `custom[fieldId]`. One getter and setter pair covers both kinds, so grouping, sorting, board columns, and search never special-case custom fields. Types: select, multiselect, checkbox, text, number, date, rating.
- **Grouping.** `groupKeys(idea, field)` returns the group or groups an idea belongs to for any field; a multi-select yields one per option, dates yield the month, empty values fall into a trailing "No <field>" group. The Ideas table and Timeline render group header rows with count, total score, and collapse. The Board renders columns from any select or checkbox field's options. Group and column choices persist per view in browser storage.
- **View state.** One `ui` object holds everything a view is: type, search, filter rules, sort, grouping, board column field, hidden columns, and timeline scale. It persists to browser storage between sessions. A saved view is a named snapshot of it stored in `data.views`; selecting one replaces `ui`, and a JSON comparison of the normalized snapshot against the saved copy drives the unsaved marker.
- **Filters.** `ui.rules` is a list of `{field, op, value}`. Operators are chosen by field type; `matchRule` evaluates one rule against an idea and all rules must pass. Status chips read and write a single Status "in" rule so the quick filter and the rule editor never disagree.
- **Formulas.** A custom field of type `formula` carries an expression string. A hand-written tokenizer and recursive-descent parser compile it to a closure once per distinct expression; there is no `eval`. Names resolve case-insensitively against the field registry at evaluation time, checkboxes read as 1 or 0, selects as option position, multi-selects as count. Evaluation depth is capped to break reference cycles, and any failure yields an empty value rather than an error in the view.
- **Lifecycle.** `links` on each idea holds `{id, type}` pairs; adding or removing one writes the inverse relationship on the other idea from a fixed inverse map. Merging moves insights, comments, labels, links and empty field values onto the target, appends the description and archives the source, so it is recoverable. `archived` is filtered out in `visibleIdeas` unless the view opts in. `rank` gives manual order: a drag renumbers every idea from the global rank sequence so a reorder inside a filtered list stays consistent.
- **View configuration.** Matrix axes, dot size and colour are field ids in `ui.matrix`, with ranges derived per field so a 1-5 rating, a 0-100 number and a formula all plot correctly. Board swimlanes reuse `groupList`, so lanes work with any field grouping already supports. `ui.cardFields` chooses card content. All of it lives in the saved-view snapshot.
- **Templates.** `data.templates` holds partial ideas. Creating from one copies a fixed key list plus custom values and records the template name in history.
- **Fiscal calendar.** `data.fiscalStart` (1 to 12) and `data.fiscalName` (`end` or `start`) live in the project file. `fiscalOf(date)` returns the fiscal quarter and year, and the Timeline's quarter headers are built from it. A January start yields plain calendar labels.
- **Export.** CSV is generated from the same filtered and sorted set the view renders, using the view's visible columns or the full set. Values are rendered as display text (option names, ISO dates, semicolon-joined lists) and RFC 4180 quoted, with a BOM for Excel. PDF is the browser's own print path rather than a bundled PDF library: a print stylesheet forces the light palette, hides the rail, toolbar and editing chrome, flattens table controls to text, and a `beforeprint` handler fills a print-only header and sets a zoom factor measured from the widest element so the table or timeline fits the page.
- **Responsive.** Below 900px the rail becomes an off-canvas drawer toggled from the toolbar, since it carries the only navigation. The timeline's sticky label column width is a CSS custom property so the scroll-to-today math and the today marker stay correct at both widths.
- **Detail panel.** Slide-over editor for a single idea, including a Fields section rendered from the custom field definitions. Field changes append a history entry describing old and new values.

Score is `reach × impact × (confidence ÷ 100) ÷ effort`, rounded to one decimal.

## Error handling

- File write failures show in the rail with the browser's error message and the data stays in browser storage.
- Import rejects files without an `ideas` array.
- Leaving the page with an unsaved file write pending prompts the browser's unload warning.

## Testing

Manual: syntax check of the script, a headless render of the list, matrix, and board views in light and dark, and exercising the sample data. No automated suite, matching the one-file scope.
