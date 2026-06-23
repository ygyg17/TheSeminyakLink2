# The Seminyak Link Bank

A simple internal link directory: a public list page and a password-gated
admin page for adding/editing/deleting links. Data is stored in a Google
Sheet. The frontend is a static site (deployable for free on GitHub Pages);
the Google Sheet is read and written through a small Google Apps Script
"Web App" that acts as a JSON API.

```
seminyak-link-bank/
├── apps-script/
│   └── Code.gs          ← paste into Google Apps Script (the backend/API)
└── docs/
    ├── index.html        ← public link list page
    ├── admin.html         ← admin login + CRUD page
    └── config.js          ← put your Apps Script Web App URL here
```

Why this split? GitHub Pages can only serve static files — it cannot run
Apps Script. So the Apps Script project stays on Google's servers (where it
can talk to your Google Sheet), and it's redeployed as a small JSON API.
The HTML pages, hosted on GitHub Pages, call that API with `fetch()`
instead of the old `google.script.run`.

---

## Part 1 — Deploy the backend (Google Apps Script)

1. Go to [script.google.com](https://script.google.com) and click **New project**.
2. Delete the default `myFunction() {}` code, then paste in the full
   contents of `apps-script/Code.gs` from this repo.
3. (Optional) Rename the project, e.g. "Seminyak Link Bank API", by
   clicking "Untitled project" at the top.
4. **Connect it to a Google Sheet:**
   - Create a new Google Sheet (or use an existing one).
   - Copy its ID from the URL — the long string between `/d/` and `/edit`:
     `https://docs.google.com/spreadsheets/d/`**`THIS_PART_IS_THE_ID`**`/edit`
   - In `Code.gs`, paste that ID into the `SPREADSHEET_ID` constant near
     the top:
     ```js
     const SPREADSHEET_ID = "1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms";
     ```
   - The script will auto-create a "Links" sheet/tab with the right
     headers the first time it runs, so you don't need to set that up by
     hand.
5. **Set your admin password.** Still in `Code.gs`, change:
   ```js
   const ADMIN_USER = "yogi";
   const ADMIN_PASS = "123";
   ```
   to something real. (See the security note at the bottom — this is a
   very lightweight login, fine for an internal/low-stakes tool but not a
   strong security boundary.)
6. **Deploy as a Web App:**
   - Click **Deploy → New deployment** (top right).
   - Click the gear icon next to "Select type" and choose **Web app**.
   - Description: anything, e.g. "v1".
   - **Execute as:** `Me`.
   - **Who has access:** `Anyone`. (This is required so your GitHub Pages
     site, which runs in visitors' browsers, can call it. It does *not*
     expose your Sheet directly — only the specific functions in
     `Code.gs` are reachable, and only the admin actions require the
     password.)
   - Click **Deploy**.
   - The first time, Google will ask you to **authorize** the script —
     click through the consent screens (you'll see an "unverified app"
     warning since this is your own personal script; click **Advanced →
     Go to [project name] (unsafe)** to proceed — this is expected for
     scripts you write yourself).
7. Copy the **Web app URL** shown after deployment. It looks like:
   ```
   https://script.google.com/macros/s/AKfycb.../exec
   ```
   Keep this tab open — you'll need this URL in the next part.

**Note:** Any time you edit `Code.gs` later, you need to create a **new
deployment version** (Deploy → Manage deployments → ✏️ → New version) for
the changes to go live. The Web App URL stays the same across versions.

---

## Part 2 — Put the frontend on GitHub

1. Create a new repository on GitHub (e.g. `seminyak-link-bank`). It can
   be public or private — GitHub Pages works with both (private repos
   need a paid plan for Pages, so public is simplest if you're on the
   free tier).
2. Upload these three files from the `docs/` folder of this project into
   your repo, keeping them inside a folder called `docs/`:
   - `docs/index.html`
   - `docs/admin.html`
   - `docs/config.js`

   Easiest way if you're not comfortable with git commands: on the repo
   page, click **Add file → Upload files**, drag in the three files (make
   sure GitHub puts them under a `docs` folder — you can type `docs/` as
   part of the path when uploading, or create the folder first by
   uploading a file named `docs/index.html` directly).

   If you do use git from a terminal:
   ```bash
   git clone https://github.com/YOUR_USERNAME/seminyak-link-bank.git
   cd seminyak-link-bank
   mkdir docs
   # copy index.html, admin.html, config.js into docs/
   git add docs
   git commit -m "Add static frontend"
   git push
   ```

3. **Edit `docs/config.js`** (directly in GitHub's web editor, the pencil
   icon, or locally) and replace the placeholder with the Web App URL you
   copied in Part 1:
   ```js
   window.API_URL = "https://script.google.com/macros/s/AKfycb.../exec";
   ```
   Commit/save the change.

4. **Turn on GitHub Pages:**
   - In your repo, go to **Settings → Pages**.
   - Under "Build and deployment" → "Source", choose **Deploy from a
     branch**.
   - Branch: `main` (or whichever branch you pushed to), folder: `/docs`.
   - Click **Save**.
   - GitHub will give you a URL after a minute or two, typically:
     ```
     https://YOUR_USERNAME.github.io/seminyak-link-bank/
     ```

5. Visit that URL — you should see the link list page. Click "Admin
   Login" (bottom of the page) to go to `admin.html` and log in with the
   username/password you set in `Code.gs`.

---

## Updating links going forward

- **Public page:** anyone with the GitHub Pages link can view and copy
  links — no login needed.
- **Admin page:** `https://YOUR_USERNAME.github.io/seminyak-link-bank/admin.html`
  — log in to add, edit, or delete links. Changes write directly to your
  Google Sheet and show up immediately on the public page.
- You can also just edit the Google Sheet directly (add a row with the
  same 5 columns: ID, Name, Link, Category, CreatedAt) — the site reads
  straight from it.

---

## Security note (please read)

This setup is intentionally simple and matches what the original Apps
Script version did, but it's worth understanding the trade-offs before
relying on it for anything sensitive:

- The admin password check happens by sending the password to your Apps
  Script, which is fine, but the password itself lives in plain text in
  `Code.gs`. Anyone with edit access to the Apps Script project can read
  it.
- The "login" on the static site is just a `sessionStorage` flag in the
  visitor's browser after a successful password check — it's not a secure
  session token. It's enough to keep casual visitors from finding the
  admin page, but a technically determined person could call the
  `addLink` / `updateLink` / `deleteLink` API actions directly without
  logging in first, since the API itself doesn't currently require a
  password on every write call.

If this ever needs to be hardened (e.g. it stops being purely internal),
the next step would be having the Apps Script check a password or token
on *every* write action, not just at login — happy to help set that up if
it becomes relevant.

---

## Local preview (optional)

You can open `docs/index.html` directly in a browser to preview styling,
but `fetch()` calls to Google Apps Script work better served over
`http://`/`https://` rather than `file://`. A quick local server:

```bash
cd docs
python3 -m http.server 8000
# then open http://localhost:8000
```
