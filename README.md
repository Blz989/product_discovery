# Discovery Desk

A single-file, no-server replacement for Jira Product Discovery. Open `index.html` in a browser and you have an ideas backlog with scoring, a status board, an impact-vs-effort matrix, a dated timeline roadmap, and insights attached to each idea.

## Run it

1. Open `index.html` in Chrome or Edge. Nothing to install, no build step.
2. Click **Save as** in the left rail and put the `.json` project file in your Box or Google Drive folder. The app writes to that file automatically after every change.
3. Next time, open the page and click **Reconnect** (or **Open file**). The browser remembers the file.

### Or use a folder instead of a file

**Save as folder** writes the project as a directory rather than one document:

```
Your Roadmap/
  project.json          name, key sequence, fiscal settings
  config/               fields, statuses, buckets, views, templates
  ideas/IDEA-12.json    one file per idea
  assets/               images, once rich text lands
```

**Open folder** reads one back. An empty folder is seeded from whatever is currently open, so
switching over is one click and your existing `.json` file stays untouched as a backup.

Why bother: in file mode every keystroke rewrites the whole document, which is fine for text and
becomes painful once descriptions carry screenshots. In folder mode a field edit writes one small
file. The rail reports exactly what it wrote, so you can watch the difference.

Both modes stay available and you can move between them. File mode remains the simplest thing that
works, and a folder is easier to sync, diff and back up per idea.

Safari and Firefox cannot write to a local file from a web page. There the app keeps everything in browser storage and you use **Export JSON** / **Import JSON** to move data around.

The first launch loads a sample project so the views have something to show. Delete those ideas or import your own file.

## What it does

| View | What you get |
| --- | --- |
| Ideas | Sortable table with inline editing of status, impact, effort, confidence, reach, and roadmap bucket |
| Board | Kanban by status. Drag cards between columns |
| Matrix | Impact vs effort scatter. Dot size is reach, color is status. Quadrant counts and top five by score |
| Timeline | Dated roadmap for one fiscal year at a time, fitted to the pane. Each idea with a start and target date is a bar. Drag a bar to move it, drag an edge to change one date. Today marker, plus lists of anything scheduled outside the year or not scheduled at all |
| Buckets | Now / Next / Later / Unplaced columns for coarse planning. Drag to move |

Every idea has a detail panel with description, owner, labels, ServiceNow demand reference, start and target dates, your custom fields, insights (text, source link, weight 1 to 3), comments, and a change history.

## Fields and grouping

Click **Manage fields** in the rail. The editor covers your own fields and the two built-in choice fields, Status and Bucket, on the same footing: rename the field, rename or recolour any option, reorder them, and add or delete options. Deleting an option tells you how many ideas it will move and reassigns them rather than leaving them stranded. You can rename Status and Bucket themselves, so Bucket can become Horizon and the label follows through the table, the panel, filters, and grouping.

Add your own fields in the same place. Types: single select, multi select, checkbox, text, number, date, rating 1 to 5, and formula.

**Formula** fields compute a number from other fields. Reference a field by name in braces, for example:

```
{Impact} * 2 + {Confidence} / 25 - {Effort} + if({Committed}, 2, 0)
```

Arithmetic, parentheses, and the functions `min`, `max`, `round`, `abs`, `sqrt`, and `if(condition, a, b)` are supported. A checkbox is 1 or 0, a single select is the option's position in its list, a multi select is the number of options chosen, and the built-in Score and Insight weight are available by name. Formulas are read-only, sortable, groupable, and filterable. The editor shows a message if a formula does not parse or names a field that does not exist. Options for selects are one per line. Renaming a field or editing its options keeps existing values; deleting a field removes its values from every idea.

Custom fields appear as editable columns in the Ideas table and as inputs in the idea panel. Search matches their values.

**Group by** in the toolbar works on the Ideas table and the Timeline, and it lists every field, built in or custom: Status, Bucket, Owner, Labels, Impact, Effort, dates, plus anything you added such as Domain or Committed. Group headers show a count and total score and click to collapse. A multi-select puts an idea under every option it carries. Dates group by month.

The **Board** has a "Columns" selector that switches its columns to any single select or checkbox field, so you can drag ideas between Domains or between Committed yes and no just as you drag them between statuses.

Grouping and column choices are remembered per view.

## Templates

**Templates** in the rail lists your idea templates. Create one from any idea with **Save as template** in the idea panel: it captures the description, status, bucket, impact, effort, confidence, reach, owner, labels and custom field values. When you add an idea, the **Start from** picker applies a template so the skeleton is already filled in. Templates live in the project file.

## Links, merging, archiving, and manual order

- **Linked ideas** in the panel connects two ideas with a typed relationship: relates to, duplicates, blocks, blocked by, parent of, child of. The matching inverse is written onto the other idea automatically, and removing a link clears both sides. Click a linked idea to jump to it.
- **Merge into…** folds a duplicate into another idea. Insights, comments, labels, links and any field the target left empty move across, the description is appended, and the source is archived and linked back as a duplicate. Nothing is deleted, so a merge can always be read back in the history.
- **Archive** takes an idea out of every view without deleting it. Archived ideas come back with **Show archived ideas** in the Filter dialog, which saved views remember, and they carry an archived pill in the list.
- **Manual order** is a `#` column, hidden by default. Show it from **Columns**, sort by it, then drag rows to reorder. New ideas go to the top.

## Configuring the views

- **Matrix**: the **Axes** button sets the horizontal and vertical fields, what dot size means, and what drives colour. Any number, rating or formula field works on an axis, and any single select or checkbox can drive colour. Axis ranges, ticks, quadrant labels and the view title follow whatever you pick, so Confidence against a weighted score reads as well as the default impact against effort.
- **Board and Buckets**: **Columns** picks the field the columns come from, **Lanes** adds optional swimlanes from any choice field, and **Cards** picks which fields appear on each card.
- All of it is captured when you save a view.

## The timeline window

The timeline shows **one fiscal year at a time**, defaulting to the current one, so a roadmap you print or hand over does not trail into last year's quarters. The picker at the top left switches between **Current fiscal year** (which follows the calendar as time passes), any specific year your ideas touch, and **All dates**.

The chart always fits the pane, so the four quarters are fully visible with no sideways scrolling. Only **All dates** over a very long range falls back to scrolling, and a Jump to today button appears when it does.

An idea that starts before the window or ends after it is still shown, clipped at the edge with a squared-off end; its full dates are in the row label and the tooltip. Dragging a clipped bar still moves the real dates. Ideas scheduled entirely in another year are listed under **Outside FY27** below the chart rather than silently disappearing, and date-less ideas stay under **Unscheduled**.

The chosen year is captured when you save a view.

## Fiscal year

Under the project name in the rail, set the month your fiscal year starts. The Timeline's quarter headers follow it, so a July start shows FY27 Q1 for July to September 2026. A second control picks whether the fiscal year is named by its ending calendar year (FY27 for July 2026 to June 2027, the default) or its starting year. Both settings are saved in the project file. Leave the start at January for plain calendar quarters.

**Score** is RICE style: `reach × impact × (confidence ÷ 100) ÷ effort`. Impact and effort are 1 to 5, confidence is a percentage, reach is any number you choose (people, transactions, hours).

## Filters, columns, and saved views

**Filter** in the toolbar opens a rule editor. Each rule is a field, an operator, and a value, and every rule must match. Any field works, built in or custom: selects use "is any of" and "is none of", numbers and ratings and formulas use comparisons, text uses contains, dates use before, after, and on, and every type can test empty or not empty. The status chips under the toolbar are a shortcut for a Status "is any of" rule. Active rules show as chips you can remove one at a time.

**Columns** on the Ideas table hides or shows any column, including custom fields.

**Save as view** captures the current view type, search, filters, sort, grouping, board columns, hidden columns, and timeline scale under a name and optional description. Saved views list in the rail and are stored in the project file, so anyone opening the file gets them. Changing anything while a saved view is active shows an unsaved marker and a **Save changes** button. **Edit view** renames, describes, or deletes it. Clicking one of the base views in the rail leaves the saved view but keeps your filters so you can keep exploring.

Search applies on top of filters in every view and covers titles, descriptions, labels, owners, insight text, comments, and custom field values.

## Keyboard

| Key | Action |
| --- | --- |
| `N` | New idea |
| `/` | Focus search |
| `Esc` | Close panel or dialog |
| `↑` `↓` or `J` `K` | Previous / next idea while the panel is open |

## Data file

Plain JSON, one object with `name`, `seq`, `statuses` and `buckets` (the editable choice lists), `fieldNames` for renamed built-in fields, a `fields` array of custom field definitions, a `views` array of saved views, a `templates` array, and an `ideas` array. Each idea carries its fields plus `insights`, `comments`, and `history` arrays. Dates are `start` and `end` as `YYYY-MM-DD` strings; leave them empty for unscheduled ideas. Custom values live under `custom` keyed by field id, links under `links`, and `archived` plus `rank` carry lifecycle and manual order. You can edit the file by hand or feed it to another tool. A file saved by an earlier version still loads: missing statuses, buckets, templates, links, ranks and archive flags are filled in with defaults on open.

## Export

**Export** in the toolbar covers the view you are looking at, with the filters and sort you have applied.

- **CSV** downloads the filtered, sorted ideas. Choose **Visible** to match the columns you have showing, or **All fields** to include everything including custom and formula fields. Two checkboxes add the full description text and the insight text. Multi-value cells (labels, multi selects) join with a semicolon, dates are `YYYY-MM-DD`, and the file carries a byte order mark so Excel opens it as UTF-8.
- **PDF** opens your browser's print dialog. Pick **Save as PDF** as the destination and keep Landscape. Any view prints, including the timeline, board, and matrix. The page gets a header with the view name, description, filter summary, idea count, and date, then the rail, toolbar, and editing controls drop away and wide content is scaled to fit the page. Printing is always on a light background even if you use dark mode.
- **Project JSON** is the same full backup as Export JSON in the rail.

The filename is built from the project, the view, and today's date.

## Phones and tablets

Below 900px the left rail becomes a drawer. Tap the menu button at the top left to reach the view switcher, saved views, fiscal year settings, field manager, project file controls, and the theme toggle. Picking a view closes the drawer. The toolbar reflows so the view name gets its own row, and the Ideas table, board, and timeline scroll sideways within their own containers.

One thing does not work by touch: dragging cards between board columns, dragging rows to reorder them manually, and dragging bars on the timeline all rely on a mouse. On a phone, tap a card or a row to open the idea and change its status, bucket, or dates there instead.

## Theme

Auto, Light, and Dark toggle in the rail. Auto follows the operating system.

## Not included

Multi-user editing and Jira Software integration. The ServiceNow demand field is free text so you can paste a demand number or a link. It is stored under the `delivery` key in the project file.

---

Extracted from the `apps/product-discovery` directory of `Blz989/claude-skills` so the tool has its
own history and issues. That repository is a Claude plugin marketplace and this was the only
application in it.
