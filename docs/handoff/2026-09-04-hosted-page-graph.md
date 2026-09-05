# Handoff: hosted page plus Microsoft Graph, for when SPFx is not available

Companion to `2026-09-04-multi-user-rearchitecture.md`. That document recommends an SPFx web part
backed by SharePoint lists. **This one covers the same goal when an SPFx package cannot be approved
in the tenant.** Read the other one first for the data model, the list schema, the concurrency
model and the code seam; none of that is repeated here. Everything below is what changes.

There is a runnable probe for the one risky assumption:
`spike/embed-probe.html`. Run it before writing any code. Section 2 says how.

---

## 1. What this path is

The existing single-file app, hosted on an internal HTTPS host, embedded in a SharePoint page with
the Embed web part, authenticating users itself with MSAL and reading and writing SharePoint lists
through Microsoft Graph.

```
 Internal host (already domain-whitelisted, HTTPS)
   index.html  +  msal-browser.js
        │ ssoSilent, falling back to a popup on a click
        ▼
   Entra ID  →  access token for Microsoft Graph
        │
        ▼
   graph.microsoft.com/v1.0/sites/{siteId}/lists/{listId}/items
        │  PATCH carries If-Match: <etag>  →  412 on conflict
        ▼
   SharePoint lists in the team's site
```

### What it buys, versus SPFx

- **No build chain.** The app stays the single HTML file it is today. No node toolchain, no
  bundler, no `.sppkg`.
- **The app keeps its own viewport.** An iframe gives it a window, so the existing `100vh` grid,
  fixed rail and slide-over panel keep working unchanged. An SPFx web part renders into a page zone
  and would have needed that layout reworked.

### What it costs

- **You write the authentication.** MSAL, an Entra app registration, token acquisition and renewal.
  SPFx would have handed you a token from the page context for free.
- **You still need an admin,** just a different approval: Entra consent for a Graph Sites
  permission instead of an app catalog deployment.
- **You host and operate the page.**
- **It may simply not work embedded.** See the next section. This is the real risk.

---

## 2. Run the probe first. Everything depends on it.

MSAL caches tokens in `localStorage` or `sessionStorage`. In a sandboxed iframe, reading
`window.localStorage` throws a `SecurityError` rather than returning nothing, which means MSAL
cannot function at all. This project already hit that exact failure once, when the app rendered as
a blank page inside a sandboxed preview frame. If SharePoint's Embed web part sandboxes your frame,
this whole path collapses and no amount of code fixes it.

`spike/embed-probe.html` answers the question in a few minutes. It is a
single file with no dependencies until you ask it to load MSAL. It checks the frame context, the
origin, secure context, Web Crypto, all three storage mechanisms, third-party cookies, whether
`graph.microsoft.com` is reachable from the origin, and then, on a click, a real MSAL sign-in and
real Graph calls. It prints a plain-text report to copy.

**How to run it:**

1. Put it on the internal host, over HTTPS.
2. Open it directly as a top-level page. This is the baseline; everything should pass except the
   sign-in tests until an app registration exists.
3. Embed **the same URL** in a SharePoint page with the Embed web part. Run it again. This result
   is the one that decides the approach.
4. Once an app registration exists, fill in the client and tenant IDs and click **Sign in and test
   Graph**. Optionally give it a site such as `contoso.sharepoint.com:/sites/FinanceTech` to prove
   the data path end to end.

**Reading the result:** if the embedded run reports an opaque origin or storage failures, stop and
take the fallback below. If it reports blocked third-party cookies but storage works, that is fine
and expected; it means the first sign-in each session uses a popup instead of being silent.

### The fallback if the probe fails

Put a prominent link on the SharePoint page that opens the app in its own tab. The app is then a
top-level page: MSAL behaves normally, no sandbox, and the layout it already expects. You lose
in-page rendering and keep everything else. This is a user-experience compromise, not an
engineering one, and it is a much better outcome than discovering the problem after building.

---

## 3. Two platform facts that shape the design

- **Cross-origin calls to SharePoint's own `/_api` are blocked.** Since SharePoint 2016, CORS
  requests to the SharePoint REST API are refused unless made through the OAuth app models, so a
  browser page on another domain cannot use it. **Use Microsoft Graph**, which is the supported
  path for browser applications.
- **Graph preserves the concurrency model.** `PATCH /sites/{siteId}/lists/{listId}/items/{id}/fields`
  honours the `If-Match` header and returns **412 Precondition Failed** when the ETag does not
  match. Everything the companion document says about optimistic concurrency, 412 recovery and the
  ban on `If-Match: *` applies unchanged.

---

## 4. Access setup

- One Entra app registration. Platform: **single-page application**. Redirect URI: the exact origin
  and path of the hosted file. The probe prints the value it expects, so register what it prints.
- Permission: prefer **`Sites.Selected`** with delegated access, granted per site, over
  `Sites.ReadWrite.All`, which grants the app every site in the organization. `Sites.Selected`
  supports delegated access, and it is a far easier request to justify to an administrator.
- `User.Read` is enough to prove sign-in works, and is usually consentable by the user without an
  admin. **Use that to validate the auth path before the permission conversation starts.** The
  probe is staged that way deliberately.

---

## 5. Code changes, in the order to do them

The renderers, filter engine, formula parser, grouping, saved views, CSV and print path all operate
on the in-memory `data` object and do not care where it came from. None of them change.

**Phase 1 — the repository seam. Ships on its own, no backend involved.**
As described in the companion document: replace whole-document saves with per-entity operations
(`saveIdea`, `deleteIdea`, `saveField`, `saveView`), keep the existing file writer as the first
implementation, make `commit()` say what changed and become async. Verify against the current build
with no behaviour change. Do this even if the rest is never built; it is a genuine improvement.

**Phase 2 — MSAL and the Graph repository.**
Add MSAL, then a second repository implementation. Resolve the site once with
`GET /sites/{hostname}:/{server-relative-path}` and cache the site and list ids. Then it is ordinary
REST against the endpoints in section 3.

**Phase 3 — concurrency and freshness.**
Hold each idea's ETag alongside it in memory. Send `If-Match` on every PATCH. On 412, refetch and
retry silently when the other edit touched a different field, and prompt only when both edits hit
the same field. There is no server, so SharePoint webhooks are unavailable; poll the lists filtered
on `lastModifiedDateTime` greater than the last check, every 30 to 60 seconds, and merge changed
items into `data`.

**Phase 4 — identity and permissions.**
`GET /me` for the current user, stamp it onto history entries so the audit trail names people, and
hide editing affordances when the user only holds read access to the list.

**Keep the local-file writer as a selectable mode.** It becomes the offline fallback and satisfies
the export requirement in the companion document for free.

---

## 6. Extra risks specific to this path

- **Token renewal inside an iframe.** MSAL renews silently using a hidden iframe. Your page is
  already in one, so that is a nested iframe, which is where silent renewal is most fragile. Plan
  for renewal failing and falling back to an interactive prompt, and make sure that failure does
  not lose unsaved edits.
- **Popups from an iframe** need a user gesture and may be blocked outright depending on the
  sandbox flags. The probe exercises this.
- **The MSAL script must load.** If the network blocks the public CDN, host `msal-browser.min.js`
  beside the app. The probe takes the script URL as an input so this can be tested quickly.
- **Content Security Policy.** If the internal host sets one, it must allow
  `login.microsoftonline.com` and `graph.microsoft.com` for connect and frame directives.

---

## 7. What could not be verified from here

The probe's environment checks were exercised in three contexts, top-level, a same-origin iframe
and a sandboxed iframe, and behave correctly in each. **The MSAL and Graph paths were not run**,
because the development sandbox has no Microsoft tenant and blocks outbound calls to
`graph.microsoft.com`. Treat the sign-in portion of the probe as unverified code that is expected
to work, not as tested code. Expect to fix small things on first contact with a real tenant.

## Sources for the platform facts cited above

- [Fixing cross domain Ajax calls to SharePoint REST services](https://techcommunity.microsoft.com/t5/microsoft-sharepoint-blog/fixing-issue-in-making-cross-domain-ajax-call-to-sharepoint-rest/ba-p/510001)
- [SharePoint REST and Microsoft Graph](https://learn.microsoft.com/en-us/sharepoint/dev/apis/sharepoint-rest-graph)
- [Update listItem, Microsoft Graph v1.0](https://learn.microsoft.com/en-us/graph/api/listitem-update?view=graph-rest-1.0)
- [Microsoft Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference)
- [Controlling app access to SharePoint sites with Sites.Selected](https://practical365.com/restrict-app-access-to-sharepoint-sites/)
