# Embed probe

`embed-probe.html` answers one question before any code gets written: can a page hosted on your own
domain, embedded in a SharePoint page, sign a user in and read SharePoint lists through Microsoft
Graph?

It matters because MSAL caches tokens in browser storage, and a sandboxed iframe makes reading that
storage throw rather than return nothing. If SharePoint's Embed web part sandboxes the frame, the
embedded approach cannot work at all, and it is much cheaper to learn that now.

## Running it

1. Put the file on your internal HTTPS host.
2. Open it directly as a top-level page. That is the baseline.
3. Embed the same URL in a SharePoint page with the Embed web part and run it again. **This is the
   result that decides the approach.**
4. Once an Entra app registration exists, fill in the client and tenant IDs and click **Sign in and
   test Graph**. Add a site such as `contoso.sharepoint.com:/sites/FinanceTech` to test the data
   path too.

Configuration is kept in the page URL, so a configured probe can be bookmarked and shared.

## Getting the values for the sign-in section

The greyed text in those fields is placeholder hint text, not values. You need an Entra app
registration first, which takes about five minutes.

1. [entra.microsoft.com](https://entra.microsoft.com) → **Applications** → **App registrations** →
   **New registration**.
2. Name it something obvious, for example `Discovery Desk probe`.
3. Supported account types: **Accounts in this organizational directory only**.
4. Redirect URI: switch the platform dropdown to **Single-page application (SPA)** and enter the
   probe's URL with no query string, for example
   `https://intranet.yourco.com/tools/embed-probe.html`.
5. Register. Do **not** add a client secret. SPA registrations use PKCE, and a secret would be
   wrong here.

| Field | Where it comes from |
| --- | --- |
| Application (client) ID | The registration's **Overview** page |
| Directory (tenant) ID | Same Overview page, directly beneath it |
| SharePoint site | Derived from the site URL: `https://contoso.sharepoint.com/sites/FinanceTech` becomes `contoso.sharepoint.com:/sites/FinanceTech` |
| MSAL script URL | Leave as-is unless your network blocks the CDN |

### Test in two stages

Leave the site field **blank** on the first run. That path needs only `User.Read`, which a new
registration gets by default and which users can normally consent to themselves. If sign-in and
"read my profile" go green, authentication works and the hard part is proven without needing anyone
else.

Only then add the site. That needs a Sites permission under **API permissions** → **Add a
permission** → **Microsoft Graph** → **Delegated**, and will almost certainly need admin consent.
Ask for **`Sites.Selected`** rather than `Sites.Read.All` where possible: it is granted per site
rather than across the whole tenant, which is a much easier request to justify.

### Three things that commonly go wrong

- **Registered as "Web" rather than "Single-page application."** The error says cross-origin token
  redemption is permitted only for the Single-Page Application client type. Fix it under
  **Authentication** by moving the redirect URI to the SPA platform.
- **Redirect URI mismatch.** It must match exactly: scheme, host, full path, no trailing query.
  After you click sign-in the probe prints the exact value it sent in the "Initialise MSAL" row;
  compare against that.
- **Your tenant may block app registration by ordinary users.** If **New registration** is greyed
  out, that is a tenant policy. Hand an admin the five settings above; it is a two-minute job.

## Reading the result

| Result | Meaning |
| --- | --- |
| Opaque origin, or storage failures | The frame is sandboxed. Embedding will not work; open the app in its own tab instead |
| Cookies blocked, storage fine | Normal. The first sign-in each session uses a popup rather than being silent |
| All green, sign-in succeeds | The embedded path is viable |

Everything it does is a read. It writes nothing to your tenant.

## Status

The environment checks are tested in three contexts: top-level, a same-origin iframe and a
sandboxed iframe. The MSAL and Graph portions are **untested**, because the machine this was built
on has no Microsoft tenant and cannot reach `graph.microsoft.com`. Expect to fix small things on
first contact with a real tenant.

Background and the full plan: `docs/handoff/2026-09-04-hosted-page-graph.md`.
