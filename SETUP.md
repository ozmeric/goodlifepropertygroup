# Good Life Property Group — BuildFlow Setup

Done-for-you project management PWA for **Good Life Property Group** (cash home buyer · Richmond, VA).
Stack: single-file React PWA (GitHub Pages) → Cloudflare Worker → Google Apps Script → Google Sheets.

```
goodlife-buildflow/
├── project-management/        ← upload this folder to GitHub Pages
│   ├── index.html             ← the app (CONFIG block at the top)
│   ├── manifest.json, sw.js   ← PWA
│   ├── icon-192.png, icon-512.png
│   └── buildflow-logo.png     ← required for the header (don't rename)
├── backend/
│   ├── Code.gs                ← Apps Script backend
│   └── worker.js              ← Cloudflare Worker proxy
└── SETUP.md
```

The app runs in **DEMO mode** (sample data saved in the browser only, any PIN works) until `API_URL` points at a real Worker.

---

## 1 · Google Sheet + Apps Script (≈10 min)

1. In the client's Google account create a blank Sheet — e.g. **"Good Life BuildFlow"**. Copy the ID from the URL (`/d/<THIS_PART>/edit`).
2. **Extensions → Apps Script**. Replace `Code.gs` with `backend/Code.gs`.
3. At the top set `SHEET_ID` and, optionally, `NOTIFY_EMAIL` (gets an email per website lead).
4. **Project Settings → Script Properties → Add**: `APP_PIN` = a **temporary PIN** for the client (4–8 digits). Send it to them separately.
5. Select `setup` in the function dropdown → **Run** → approve permissions. This creates the tabs:
   `Projects · Leads · Milestones · Vendors · Management · General Expenses`
   (each project later gets `<name>`, `<name> - Tasks`, `<name> - Quotes` tabs).
6. **Deploy → New deployment → Web app** · Execute as **Me** · Who has access **Anyone** → copy the `/exec` URL.

> After editing Code.gs later: **Deploy → Manage deployments → Edit → Version: New** (keeps the same URL).

### PIN flow
- **First login:** the client enters the temporary PIN and must choose their own 4–8 digit PIN before the app opens. Repeated digits (`1111`) and sequences (`1234`) are refused.
- After that, `APP_PIN` is deleted and only a salted SHA-256 hash is stored (`APP_PIN_HASH`, `APP_PIN_SALT`). Nobody, including you, can read their PIN.
- **Change PIN anytime:** sidebar → **PIN** on desktop, **Account → Change PIN** on phones. Requires the current PIN.
- **Forgot PIN:** add `APP_PIN` again in Script Properties with a new temporary PIN (or put it in `setTemporaryPin()` and Run). A temporary PIN always overrides the old one, and anyone already signed in is sent back to the PIN screen on their next action.
- **Brute-force guard:** after 20 wrong PINs in 15 minutes, logins pause for 15 minutes.

## 2 · Cloudflare Worker (≈3 min)

1. Cloudflare → Workers & Pages → Create Worker, e.g. `goodlife-buildflow`.
2. Paste `backend/worker.js`, set `APPS_SCRIPT_URL` to the `/exec` URL → Deploy.
3. Once the app's domain is final, set `ALLOWED_ORIGINS` to it (and the marketing site's domain if using the lead form).

## 3 · Frontend (≈5 min)

1. In `project-management/index.html` set:
   ```js
   const API_URL = "https://goodlife-buildflow.<your-subdomain>.workers.dev/";
   ```
   (Removing `YOUR-` from the URL switches DEMO mode off.)
2. Upload the whole `project-management/` folder to the GitHub Pages repo — including `buildflow-logo.png`.
3. Suggested URL: `goodlife.usebuildflow.io` (CNAME under the usebuildflow.io wildcard) or `goodlifepropertygroup.com/project-management`.
4. On the phone: open the URL → Share → **Add to Home Screen**.

## 4 · Website → Deal Pipeline (optional, recommended)

goodlifepropertygroup.com's "Get a cash offer" form can drop leads straight into **Company → Deal Pipeline** (stage *New Lead*, source *Website*) — no PIN needed for this one action. Add to the form's submit handler:

```html
<input type="text" name="website" style="display:none" tabindex="-1" autocomplete="off"> <!-- honeypot -->
<script>
async function sendLead(form) {
  const f = Object.fromEntries(new FormData(form));
  const res = await fetch("https://goodlife-buildflow.<your-subdomain>.workers.dev/", {
    method: "POST",
    body: JSON.stringify({ action: "publicLead", lead: {
      name: f.name, phone: f.phone, email: f.email,
      address: f.address, situation: f.situation, message: f.message, website: f.website } })
  });
  return res.json(); // { success: true } or { error: "..." }
}
</script>
```

The backend requires a phone or email, plus a name or address, and ignores submissions that fill the hidden `website` field.

---

## What's configured for Good Life

| Area | Setup |
|---|---|
| Branding | Deep green `#0f2e22` + gold `#d6a43c`, "G" mark, (804) 589-5300 in sidebar. Change in `CONFIG` (+ `theme` tokens). |
| Project types | Fix & Flip · Wholesale / Assignment · Buy & Hold · Rental Rehab · New Construction |
| Statuses | Under Contract → Active → Listed → Sold (archived) · On Hold |
| Deal Pipeline | New Lead → Contacted → Appointment Set → Offer Made → Under Contract → Closed / Dead; situations mirror the website (inherited, divorce, foreclosure, tired landlord, fire damage…). Live **MAO = ARV × 70% − repairs**; "Make Project" converts a contracted lead into a project. |
| Expenses | 20 categories. Acquisition, Closing, Carrying and Selling costs don't count against the rehab budget. |
| Dashboard | Purchase, rehab budget/spent/left, loan, ARV, projected profit & ROI (8% est. selling costs), spend by category, tasks, milestones. |
| Also | Tasks (tap-to-cycle status, milestone links), Quotes (accepting auto-rejects other bids and logs the expense), Planning timeline, Vendors directory, Company tasks/milestones, General Expenses. |

Tunable in `CONFIG`: `maoPercent`, `sellingCostPct`, all category lists, labels, colors.

## Before handing off
- [ ] Confirm brand colors against the logo (`CONFIG.primaryColor`, `accentColor`, `theme`)
- [ ] Temporary `APP_PIN` set, `setup()` run, web app deployed as **Anyone**
- [ ] Log in once with the temporary PIN to confirm the "choose your PIN" step, then reset `APP_PIN` to a fresh temporary PIN for the client
- [ ] Worker URL in `API_URL`; DEMO badge gone after reload
- [ ] `buildflow-logo.png` uploaded next to `index.html`
- [ ] Test: add project → expense → task → quote accept; submit a test website lead
