# Handoff: multi-user re-architecture for Discovery Desk

Written 2026-09-04 at the end of the session that built the single-user tool. It exists so the
next agent does not have to re-derive the decisions, the constraints, or the shape of the code.

**Read these first, in order:** this file, then `README.md` (what the tool
does, from the user's side), then `docs/specs/2026-09-03-product-discovery-design.md` (how it is
built). Do not duplicate those here; they are maintained and current.

---

## 1. Where things stand

Discovery Desk is a working replacement for a Jira Product Discovery seat. It is one HTML file,
`index.html`, about 1,900 lines, no build step and no dependencies beyond a
Google Fonts link. It is feature-complete for one person and in daily use.

Built and merged: ideas table with inline editing, board, impact/effort matrix, fiscal-year
timeline, Now/Next/Later buckets, user-editable fields including statuses and buckets, formula
fields, saved views, rule-based filters, idea links, merge, archive, manual ranking, templates,
CSV and PDF export, phone layout.

Deliberately not built: rich text and attachments (the owner declined), Jira Software delivery
integration (standalone by choice), insight capture from Slack/Teams/Zendesk.

Known gaps besides multi-user, in the owner's priority order: bulk edit, undo, CSV import, and
filter logic beyond AND. None of these are blocked by the re-architecture and could ship first.

---

## 2. The decision that prompted this handoff

The owner is in a Microsoft 365 / SharePoint organization and wants the team on the same backlog.
Three things were established directly with them on 2026-09-04:

| Question | Answer |
| --- | --- |
| How many people edit? | Five or more, plus read-only stakeholders |
| What can be approved in the tenant? | An SPFx package can be approved into an app catalog |
| Is a shared store acceptable instead of the local file? | Yes. A shared store is preferred. Full export must survive |

Five or more editors rules out the cheaper options that were on the table earlier in the session,
specifically a per-viewer browser-storage copy or a read-only published snapshot. This needs a
real shared store with concurrency handling.

---

## 3. Recommended target, and why

**SPFx web part on a SharePoint page, backed by SharePoint lists in the same site.**

Reasons, in order of weight:

1. **Identity is free.** An SPFx web part runs in the page's context, so the current user, their
   display name and their permissions arrive without writing any authentication. Every other
   option means building sign-in.
2. **Permissions are the platform's.** Editors versus read-only stakeholders is site and list
   permissions, not code. This matters more with five-plus people than any feature on the list.
3. **The approval exists.** The owner confirmed an SPFx package can be approved, which was the
   main blocker on this path.
4. **No hosting bill and no second system to operate.**

### Options considered and rejected

- **Dataverse.** Attractive on paper as a middle ground, and the owner raised it. Two problems.
  Full Dataverse needs premium Power Apps licensing per user, a real per-seat cost that
  reintroduces the thing this project set out to avoid. Dataverse for Teams is free with an
  Office-seeded licence but its apps only run inside Teams, so it cannot serve a SharePoint page,
  which is the stated delivery surface. Revisit only if the org already holds premium Power Apps
  licences for these users; confirm before assuming.
- **Azure Static Web Apps plus Functions.** Most flexible and the best fit if the tool ever needs
  to live outside M365. Costs money above the free tier once you need tenant-restricted sign-in,
  and you own the auth, the hosting and the database. Keep as the fallback if SPFx approval falls
  through.
- **Hosted HTML page embedded in SharePoint, talking to list APIs.** Avoids the app catalog but
  needs its own sign-in, needs the host domain allowed under HTML Field Security, and inherits the
  iframe storage limits documented in section 7. Strictly worse than SPFx once SPFx is available.
- **Keep the single JSON file on OneDrive or Box.** What exists today. Last-writer-wins with
  conflict copies. Fine for one person, unsafe for five.

---

## 4. The single most important thing to understand about the code

**Persistence is already behind a narrow seam, but it saves the whole document on every keystroke.**
That is the core of the refactor, and it is more important than the UI work.

Today:

- `data` is one global object holding the entire project: ideas, field definitions, statuses,
  buckets, saved views, templates, fiscal settings.
- `loadLocal()` reads it, `normalize()` fills in defaults and migrates older files, `persist()`
  serializes **the entire object** and writes it, `commit()` calls `persist()` then re-renders.
- Every mutation follows the same shape: change the object, then call `commit()`. See `setField`,
  `setVal`, `addHistory`, `mergeInto`, `moveRank`.

For a local file, writing everything on every change is fine. Against a shared list it is wrong on
three counts: it is slow, it overwrites other people's concurrent edits wholesale, and it makes
per-item conflict detection impossible.

**The refactor is therefore: move from document-level saves to entity-level saves.** Concretely:

1. Introduce a repository layer with per-entity operations: `saveIdea(idea)`, `deleteIdea(id)`,
   `saveField(def)`, `saveView(view)` and so on. Give it two implementations behind one interface,
   a local-file one for the existing mode and a SharePoint one.
2. Change `commit()` to take what actually changed, for example `commit({idea})`, and have the
   repository write only that. Every call site already knows which entity it touched, so this is
   mechanical rather than invasive.
3. Make the write path async and handle rejection. `commit()` is synchronous today.

Do this refactor **against the existing local-file build first**, verify nothing regresses, and
only then add the SharePoint implementation. That keeps the diff reviewable and leaves the tool
working throughout.

### The other contract worth preserving

The **field registry** in `fieldDefs()` is what makes the tool flexible: built-in fields and
user-defined fields share one descriptor `{id, name, type, options[]}`, and one getter/setter pair
(`getVal`, `setVal`) covers both. Grouping, sorting, filtering, board columns, matrix axes and
search all read through it and never special-case a field. **Do not break this.** Any storage
design that forces built-in and custom fields down different paths will unravel most of the UI.

---

## 5. Suggested SharePoint list schema

Four lists in one site. Names are suggestions.

**`DiscoveryIdeas`** — one item per idea.

| Column | Type | Notes |
| --- | --- | --- |
| Title | Single line | The idea title |
| IdeaKey | Single line | `IDEA-12`, indexed. See the note on key allocation below |
| Status, Bucket | Single line | Store the **option id**, not the label. Labels live in config and are renameable |
| Impact, Effort, Confidence, Reach, Rank | Number | |
| StartDate, TargetDate | Date | |
| Owner | Person | Use the real person field, not text; it gets you presence and filtering |
| Labels | Single line | Semicolon-joined, or a managed metadata field if the org prefers |
| SnDemand | Single line | The ServiceNow demand reference |
| Archived, MergedInto | Yes/No, Single line | |
| CustomJson | Multiple lines, plain text | All custom field values as JSON, keyed by field id |
| LinksJson, HistoryJson | Multiple lines, plain text | History is already capped at 60 entries per idea |

**`DiscoveryInsights`** and **`DiscoveryComments`** — child items with a lookup to the idea. These
are genuinely list-shaped, people will want to report on them, and they grow independently.

**`DiscoveryConfig`** — configuration, **one item per entity, not one item per category**. One
item per saved view, one per custom field definition, one per status option, one per bucket, one
per template, plus a single item for project-level settings such as the fiscal calendar. Columns:
`Kind` (`view` / `field` / `status` / `bucket` / `template` / `settings`), `EntityId`, `SortOrder`,
and `Json` holding that one entity.

The granularity matters. An earlier draft of this document put each category in a single JSON
document, which would mean that renaming a field while a colleague adds a saved view silently
destroys one of the two changes. Ideas are naturally partitioned into rows; configuration has to
be partitioned deliberately, or it becomes the contention hotspot of the whole system.

### Why custom values go in a JSON column rather than real columns

Users create and delete fields at runtime. Mirroring that as list columns means creating and
deleting columns through the API on every field change, which is slow, hits column limits, and
leaves orphaned columns behind. A JSON column keeps the field registry contract intact at the cost
of server-side querying on custom fields. That cost is acceptable here because the app already
filters, sorts and groups **client-side** over the full set, and the set is small. Revisit only if
the backlog ever passes a few thousand ideas.

### Key allocation: drop the counter

The file-based app mints keys from a shared counter, `data.seq`. With concurrent editors that is a
duplicate-key bug waiting to happen: two clients read the same value and both write `IDEA-12`.

Use SharePoint's own auto-increment item ID instead, so a new idea's key is `IDEA-{item id}`. No
coordination, guaranteed unique, and gaps after a deletion are normal and match what Jira does.
Preserve keys from imported files as-is in `IdeaKey`; only mint new keys this way.

### Concurrency in practice

There is no locking and no client-held transaction. Each write is an independent request, and
SharePoint serializes writes to a given item server-side.

- **Two people creating ideas at once**: no contention. Independent inserts into different rows.
- **Two people editing the same idea**: send the ETag you last read in an `If-Match` header.
  SharePoint returns **412 Precondition Failed** if the item changed underneath you, rather than
  overwriting. Do not send `If-Match: *`; that is last-writer-wins and is exactly the silent
  data loss requirement 4 forbids.
- **Recovering from a 412**: refetch the item. If the other edit touched a different field,
  reapply the user's change and retry without bothering them. Prompt only when both edits hit the
  same field. The app saves one field at a time, so this covers most real collisions quietly.
- **Nobody gets pushed updates.** SharePoint webhooks are server-to-server and there is no server
  here, so poll the lists filtered on `Modified` greater than the last check, every 30 to 60
  seconds, and merge changed items into `data`. Filtering on `Modified` keeps it cheap.

### The list limit that actually matters

SharePoint Online caps a **list view** at 5,000 items and that cap cannot be raised in the online
service, though a list can hold up to 30 million items. A discovery backlog is hundreds of ideas,
so ideas are never at risk. Watch the insights and comments lists over years of use, index any
column used in a filter, and page the reads.

---

## 6. Requirements the re-architecture must meet

Non-negotiable, agreed with the owner:

1. **Full export survives.** A one-click export that reproduces the complete project as the same
   JSON the file-based version uses. The owner accepted a shared store on this condition. It is
   also the migration path back out if this is ever abandoned.
2. **No feature regressions.** Use `README.md` as the acceptance checklist;
   it documents every user-facing behaviour. Pay particular attention to saved views, formula
   fields, the fiscal-year timeline and the field manager, which are the subtle ones.
3. **Existing project files import.** Users have local JSON files. `normalize()` already
   backfills anything missing from older files; reuse it rather than writing a second migration.
4. **Concurrency is handled visibly.** SharePoint list items carry an ETag. Use optimistic
   concurrency and tell the user plainly when their edit lost a race. Silent overwrites with five
   editors will destroy trust in the tool faster than any missing feature.
5. **Read-only stakeholders get a clean view.** They should not see editing affordances they
   cannot use.

---

## 7. Constraints already discovered the hard way

Do not re-learn these.

- **A `.html` file in a document library downloads rather than renders**, unless a tenant admin
  enables custom script. This is why a plain hosted file was never the answer. An SPFx web part
  sidesteps it entirely.
- **Sandboxed iframes make reading `window.localStorage` throw**, not return null. An unguarded
  read at module scope took the whole app down with a blank page. All storage access now goes
  through a guarded `store` accessor. If any of that code survives, keep the guard.
- **Sandboxed iframes also block downloads**, so export and import are disabled there.
- **The File System Access API does not work cross-origin in any iframe.** The local-file mode is
  a directly-opened-page feature only.
- **Print, not a PDF library.** PDF export goes through the browser's print path with a print
  stylesheet. It prints the real timeline rather than a redrawn approximation and adds nothing to
  the bundle. Keep this.
- **Drag and drop is mouse-only.** The board and manual reordering use HTML5 drag events, which do
  not work on touch. The timeline uses pointer events and does work. Converting the board is a
  known open task.

---

## 8. Working conventions from this session

- **One branch per change, one PR per branch**, named `claude/<topic>-0l8a8w`. Never commit to
  `main`. The owner asked for this explicitly and values reviewable, single-purpose commits.
- **PR bodies state cause, fix and testing.** For bug fixes, state the root cause plainly, not
  just the symptom.
- **Testing is headless Playwright** against the local file. The pinned browser in this
  environment is
  `/opt/pw-browsers/chromium_headless_shell-1194/chrome-linux/headless_shell`; the bundled
  Chromium binary is a version mismatch and fails to launch. Scripts live in the scratchpad, not
  the repo. Typical pattern: launch, `localStorage.clear()`, reload, drive the UI, assert on DOM
  state, capture `pageerror`.
- **Verify the old build fails before claiming a fix works.** This caught one case where a test
  harness quirk, not the app, was producing the failure.
- **Keep README and the design spec current in the same commit as the change.**

---

## 9. Suggested phasing

1. **Repository refactor on the existing build.** Entity-level saves behind an interface, local
   file implementation only. No behaviour change. Ships independently and is valuable on its own.
2. **SPFx shell.** Wrap the existing renderer in a web part. Do not rewrite the UI in React; the
   code is vanilla and self-contained, and a rewrite risks every subtle behaviour in section 6.
3. **SharePoint repository implementation** plus provisioning of the lists, ETag concurrency, and
   the import path for existing JSON files.
4. **Multi-user affordances.** Who last edited an idea, read-only mode for stakeholders, and
   whatever conflict UI testing shows is needed.
5. **Only then** the deferred single-user gaps: bulk edit, undo, CSV import.

Bulk edit and undo get materially harder after step 1 changes the save shape, so if the owner
wants them soon, consider doing them before the SPFx work rather than after.

---

## 10. Open questions to confirm before building

- Does the org already hold premium Power Apps licences for these users? If yes, Dataverse is back
  on the table and worth a fresh comparison.
- Which SharePoint site hosts this, and who administers its permissions?
- Should stakeholders outside the immediate team see it, and does anything need external sharing?
- Is there an existing SPFx build and deployment practice in the org to follow, or is this the
  first one?

## Sources for the platform facts cited above

- [List View Threshold for large lists and libraries](https://support.microsoft.com/en-us/sharepoint/lists/data-and-lists/list-view-threshold-for-large-lists-and-libraries)
- [The number of items in this list exceeds the list view threshold](https://learn.microsoft.com/en-us/troubleshoot/sharepoint/lists-and-libraries/items-exceeds-list-view-threshold)
- [About the Microsoft Dataverse for Teams environment](https://learn.microsoft.com/en-us/power-platform/admin/about-teams-environment)
- [Power Apps licensing FAQs](https://learn.microsoft.com/en-us/power-platform/admin/powerapps-licensing-faq)
- [Allow or prevent custom scripts in SharePoint sites](https://learn.microsoft.com/en-us/sharepoint/allow-or-prevent-custom-script)
