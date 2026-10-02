# JDA3 Report — Setup Reference

Two files — `index.html` and `styles.css` — plus the image assets alongside
them, backed by Firestore, no login required to view. Anyone with the link
can view and edit the report data — see the note on that trade-off further
down. Microsoft Calendar sync (optional) is the one feature that does ask a
user to sign in, but only when they choose to use it.

---

## 0. File structure

| File | What's in it |
|---|---|
| `index.html` | Page markup, plus all behaviour in an inline `<script type="module">` near the bottom — rendering, Firestore reads/writes, chart setup, Edit Report logic, Microsoft Calendar sync. Your Firebase config (`firebaseConfig`), the edit passkey (`EDIT_PASSKEY`), and the Microsoft `MS_CLIENT_ID`/`MS_AUTHORITY` are all in there. |
| `styles.css` | All styling. Linked via `<link rel="stylesheet" href="styles.css">`. |
| `sol-*.png`, `*-Logo*.png/jpg`, `TriCipta.jpg` | Image assets referenced by plain relative filename — no embedded base64 anywhere in the HTML/CSS. |

**The JS deliberately stays inline in `index.html` rather than living in its
own `.js` file.** An external module script (`<script type="module"
src="app.js">`) can't be fetched over a `file://` address at all — the
browser blocks it — so double-clicking `index.html` would show the hero and
nothing else, with no data loading. Keeping the script inline (while the CSS
stays external via `<link>`, which has no such restriction) is what lets you
still just double-click `index.html` and have the full page work, same as
before. If you ever do want the JS in its own file again, that only works
once the page is always opened over `http(s)://` — never via a raw file
path — which `vercel dev` or any local static server gives you.

---

## 1. Firestore security rules

Firebase console → **Firestore Database → Rules tab** → replace with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /report/{doc} {
      allow read: if true;
      allow write: if true;
    }
  }
}
```

Click **Publish**. This scopes open access to only the `report`
collection — nothing else in your Firebase project is exposed.

---

## 2. Your Firebase config

Already pasted into `index.html` near the top of the `<script type="module">`
section (`const firebaseConfig = {...}`).
If you ever need to point this at a different Firebase project, get the new
values from **Project settings → Your apps → (web app) → SDK setup and
configuration** and paste them in.

---

## 3. Seed the starter data

Open `index.html` in a browser. Click **Edit report** in the footer — if
Firestore's `report/current` document doesn't exist yet, you'll see a
**Load starter data** button. Click it to seed a full mock dataset (projects,
tasks, challenges, updates, S-curve).

If the document already exists but a section is empty (e.g. you cleared
Challenges), use that section's own "Load sample …" button instead — the
Challenges card has **Load sample challenges**, which only touches that
field.

---

## 4. Deploy to Vercel

No build step — this is a set of plain static files (`index.html`,
`styles.css`, images). Vercel serves the whole folder as-is.

```
npm install -g vercel
cd path/to/JDA3_dashboard
vercel
```

Follow the prompts. You'll get a live URL in under a minute. Re-run
`vercel --prod` any time you change the HTML/CSS itself — data edits made
through the "Edit report" panel go straight to Firestore and don't need a
redeploy.

---

## 5. Microsoft Calendar sync (optional)

Lets someone click **Connect Microsoft Calendar** in the edit panel and push
open tasks (that have a due date) to their own Outlook Calendar as real
events, using their organization's Microsoft 365 / Azure AD account.
Re-syncing updates the same events instead of duplicating them.

**Important constraint:** Microsoft's OAuth flow refuses to run on a
`file://` page. You must test this over `http://` or `https://` — either a
local server or your deployed Vercel URL. Opening `index.html` by
double-clicking it will show a clear error if you try to connect from there.

### 5.1 Register an app in Azure AD (Microsoft Entra ID)

You'll need access to your organization's
[Azure Portal](https://portal.azure.com/) — this usually requires being a
member of the org's Azure AD tenant; IT admin approval may be needed for the
permission step below, depending on your org's policies.

1. Go to **Azure Active Directory** (also shown as **Microsoft Entra ID**) →
   **App registrations** → **New registration**.
2. Name it something recognizable, e.g. "JDA3 Report Calendar Sync".
3. **Supported account types**: choose **Accounts in this organizational
   directory only** to restrict sign-in to your company, or **Accounts in
   any organizational directory** if people from partner orgs (IBM,
   Tridiagonal) should also be able to sync their own calendars.
4. **Redirect URI**: set the platform to **Single-page application (SPA)**
   and add every origin you'll actually open the page from, e.g.:
   - `http://localhost:5500` (or whatever port your local server uses)
   - `https://your-project.vercel.app` (your real Vercel URL)
5. After creation, copy the **Application (client) ID** from the app's
   Overview page.
6. **API permissions → Add a permission → Microsoft Graph → Delegated
   permissions** → search for and add **Calendars.ReadWrite**. If your
   org requires admin consent for this scope, click **Grant admin consent**
   (or ask an IT admin to do it) — otherwise each user will be blocked at
   sign-in with a consent error.

### 5.2 Paste the Client ID into the file

In `index.html`, find:

```javascript
const MS_CLIENT_ID = "YOUR_AZURE_AD_CLIENT_ID";
const MS_AUTHORITY = "https://login.microsoftonline.com/organizations";
```

Replace `MS_CLIENT_ID` with the Application (client) ID from step 5.1.
`MS_AUTHORITY` as `"organizations"` accepts any Microsoft work/school
account; to restrict sign-in to only your company's tenant, change it to
`"https://login.microsoftonline.com/<your-tenant-id-or-domain>"` (e.g.
`.../contoso.onmicrosoft.com`) — find your tenant ID or verified domain on
the Azure AD **Overview** page. Redeploy (`vercel --prod`) if you're
already live.

### 5.3 Testing locally without deploying

Double-clicking `index.html` gives it a `file://` address, which Microsoft
refuses. Serve it over `http://localhost` instead — any of these work, run
from the `JDA3_dashboard` folder:

```
npx serve .
```
or
```
python -m http.server 5500
```

Then open the `http://localhost:...` URL it prints, not the file path.
Make sure that exact `http://localhost:PORT` origin is registered as a
redirect URI (step 5.1) or Microsoft will reject the sign-in.

### 5.4 What actually gets synced

- Only tasks with a due date **and** a status other than "Done".
- Each task becomes an all-day Outlook Calendar event on the signed-in
  person's **default** calendar, titled `JDA3: <task name>`.
- Events are tagged internally (via a Graph extended property) so
  re-clicking "Sync" updates the same event instead of creating duplicates.
- The access token lasts about an hour; if it expires, just click **Connect
  Microsoft Calendar** again.
- Nothing syncs automatically in the background — sync only happens when
  someone is on the page and clicks the button.

---

## 6. Connect to Excel via Power Automate (SharePoint)

Lets a Power Automate cloud flow read a project's Excel workbook (in
SharePoint) and push updates into the same Firestore document
(`report/current`) the dashboard already renders from — no changes to the
UI needed. **One workbook per project** — the real one in use is
`260925 Progress Report.xlsx`, with one `<code> Daily` + `<code> S Curve`
sheet pair per project (`P2a`, `P2b`, `P3a`, `P4`, `P6`, `P7`, `CDF`).

Power Automate's own connectors can sign into your Microsoft 365 account
inside the flow designer itself — there's no Azure AD app registration or
client secret to create for this path, unlike the OneDrive/Graph-API backend
explored earlier. The one piece of custom code is a small helper endpoint
(`api/sync-report.js`), which exists only so the flow never has to build
Firestore's verbose typed-value JSON, or do its own read-modify-write merge
— it just POSTs one project's plain data and the endpoint does the rest
safely (matching by project name, replacing only that project's own tasks,
updates and S-curve, leaving every other project and every other field
untouched).

**Licensing note:** the HTTP action used below is a *Premium* Power
Automate connector. Depending on your Microsoft 365 licence, that may need
a Power Automate per-user/per-flow plan (often available as a free trial) —
check with your IT admin if it's greyed out when you try to add it.

### 6.1 The Excel ↔ website project mapping (⚠️ only P2b is confirmed)

Every sheet pair has already been restructured into real Excel Tables (see
6.2), so the workbook is ready for someone to fill in — but the mapping
below from Excel project code to the website's actual project name is only
**verified** for `P2b` (its real content — "BOKOR", "Reality Capture
Handover" — matches what's already live). The rest are educated guesses
based on numbering, since their sheets are still empty templates with no
content to check against. **Confirm these before relying on them**, and fix
the `project.name`/`projectName` value in any flow you build if wrong —
`api/sync-report.js` will refuse an unrecognised name rather than silently
create a bad entry, so a wrong guess fails loudly, it doesn't corrupt data.

| Excel code | Website project name | Confidence |
|---|---|---|
| `P2a` | `Project 2a - SAP Master Data Revitalisation` | guess (numbering) |
| `P2b` | `Project 2b - Vision Analytics for Asset Integrity` | ✅ verified |
| `P3a` | `Project 3a - AI-Driven Asset Integrity (Predictive LOPC & Sand Monitoring)` | guess |
| `P4` | `Project 4a - Maintenance & Reliability` | guess |
| `P6` | `Project 6 - Production Optimisation & Planning` | guess |
| `P7` | `Project 7/TA - Work Process Efficiency & Turnaround` | guess |
| `CDF` | `Project CDF - Contextual Data Fusion` | guess |

(`Project 2b/3a` used to be one combined project — it's now split into
`P2b` and `P3a` separately, and `Project TCO` was removed since nothing in
Excel tracks it; it stays a website-only, manually-edited entry if it ever
comes back.)

### 6.2 What's already in the workbook

Every project's sheet pair now has the same three real Excel Tables, so
there's nothing left to restructure — just fill them in:

- **`<code>_Tasks`** (on the `S Curve` sheet) — columns `R&R` (owner),
  `Spacer`, `Action Items` (task name), `Task involved`, `Plan Date Start`,
  `Plan Date End`, `Actual Date`, `Weightage`. A row counts as done once
  `Actual Date` is filled in.
- **`<code>_Updates_TAI`** / **`<code>_Updates_IBM`** (on the `Daily`
  sheet) — the two side-by-side dated bullet-log blocks, one per partner.
  Column `Weekly Update Description` + `Date`.

There's deliberately **no "Summary" sheet** — `Owner`/`Status`/`Description`
stay manually edited on the website's own Edit Report screen, never from
Excel. `Completion %` isn't typed anywhere either — the flow computes it
itself: sum `Weightage` for every `<code>_Tasks` row that has a non-blank
`Actual Date`, since each project's rows sum to 100.

A "Milestone : ..." label that used to sit inside the update-bullet block
(breaking the Table's single-header-row requirement) has been moved to a
plain note cell just above each `Updates` table — it's not part of any
Table and isn't read by the flow; the website derives "current milestone"
itself from whichever task is `In progress`.

**Still unresolved — no ready source yet:** the website's per-project
S-curve chart wants a *monthly* `{months, planned, actual}` rollup, but
`<code>_Tasks` only has per-task `Weightage` and dates, not a month-level
summary. Filling in more task rows won't populate that chart on its own —
either add a small monthly summary table to each sheet, or build the
month-by-month aggregation as Power Automate expressions. Not designed yet.

### 6.3 Deploy the helper endpoint

From this folder:

```
npm install -g vercel
vercel
```

Follow the prompts (same as section 4). Note the resulting URL — the
endpoint you'll call from Power Automate is:

```
https://<your-project>.vercel.app/api/sync-report
```

No environment variables are required (it defaults to this dashboard's
Firebase project). Test it works before building the flow:

```powershell
Invoke-RestMethod -Uri "https://<your-project>.vercel.app/api/sync-report" -Method Post -ContentType "application/json" -Body '{"project":{"name":"Project 2b - Vision Analytics for Asset Integrity","pct":9}}'
```

If that project name doesn't exist yet in the live report, the response
lists the current valid names instead of guessing.

### 6.4 Build the flow (one project as the worked example — P2b)

In [Power Automate](https://make.powerautomate.com/):

1. **Create → Scheduled cloud flow** (or "Automated," triggered instead by
   "When a row is modified" on `P2b_Tasks`, if you'd rather it react to
   edits than run on a timer).
2. **+ New step → Excel Online (Business) → List rows present in a table**,
   three times — once each for `P2b_Tasks`, `P2b_Updates_TAI`, and
   `P2b_Updates_IBM` (Location: the SharePoint site; File: `260925 Progress
   Report.xlsx`).
3. **Compute completion %** — add a **Data Operation → Filter array** on
   `P2b_Tasks`' rows where `Actual Date is not equal to` blank, and a
   **Compose** action summing that filtered array's `Weightage` values
   (`sum(...)`, an expression over the filtered array — Power Automate's
   expression editor, not a separate connector).
4. **+ New step → Data Operation → Select**, once per tasks/updates table,
   to reshape each row into the plain object `api/sync-report.js` expects
   (`{task, due, status, owner}` for tasks — `status` from an `if()`
   expression checking whether `Actual Date` is blank; `{date, text}` for
   updates, unioning the TAI and IBM lists together with `union(...)`).
5. **+ New step → HTTP.**
   - Method: `POST`
   - URI: `https://<your-project>.vercel.app/api/sync-report`
   - Headers: `Content-Type: application/json`
   - Body:

     ```json
     {
       "project": { "name": "Project 2b - Vision Analytics for Asset Integrity", "pct": <Compose output from step 3> },
       "tasksForProject": <Select output from step 4 (tasks)>,
       "updatesForProject": <Select output from step 4 (updates)>
     }
     ```

     The project `name` is typed literally, matching 6.1's table — it must
     exactly match the name already on the dashboard, the same safety
     check the earlier local script had. `scurveForProject` isn't included
     yet — see the open item in 6.2.
6. **Save**, then **Test → Manually** and confirm the dashboard's Project
   2b detail page updates. If the HTTP step errors, its response body says
   exactly what's wrong (e.g. an unrecognised project name).

### 6.5 Replicating for the other 6 projects

Once step 6.4 is proven on P2b: duplicate the flow (or parameterise it) for
each of the other 6, pointing at that project's own `<code>_Tasks` /
`<code>_Updates_TAI` / `<code>_Updates_IBM` tables and using the matching
name from the 6.1 table — **after confirming that mapping is actually
correct**, since right now only `P2b`'s is verified.

---

## A note on the open-write trade-off

Because `allow write: if true`, anyone who has the link can change the
report's data — not just view it. There's no login and no record of who
made a change. That's a reasonable trade for an early-stage internal tool
with no sensitive data, but worth keeping in mind once this is actively
shared with stakeholders: a wrong number showing up with no way to trace
who entered it is the main risk, not data exposure. Microsoft Calendar sync
doesn't change this trade-off — it's a separate, opt-in sign-in that only
grants access to that one person's own calendar events, not to the report
data itself.

## The "Edit report" passkey isn't real security

The Edit Report screen is gated behind a 4-digit passkey (`1234` by
default, set in `index.html` as `const EDIT_PASSKEY`). This is **friction,
not security** — it's a plain string checked in client-side JavaScript, so
anyone who opens the browser's dev tools and reads the page source can see
it instantly, and Firestore's rules still allow writes from anyone
regardless of the passkey (see above). Its only real job is to stop
someone from casually clicking "Edit report" and poking at fields by
accident. Once entered correctly, it stays unlocked for that browser tab
until the tab is closed (stored in `sessionStorage`) — reopening the page
in a new tab asks again. If you ever need this to be actually secure
(audit trail, real per-person access control), that requires Firebase
Auth and rewritten Firestore rules, not a passkey.
