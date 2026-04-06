# Crash Course — Hosting & Sync Setup

Host study apps on GitHub Pages with cross-device progress sync via Google Sheets.

## Architecture

```
GitHub Pages (static hosting)
  └── serves HTML study apps directly (no processing, no sanitization)

Google Sheets + Apps Script (sync API only)
  ├── GET  ?action=load&key=gke-crash-course  → reads progress from Sheet
  └── POST ?key=gke-crash-course              → writes progress to Sheet
```

**Why this split?** The study apps are standalone HTML files — they don't need a backend to run. GitHub Pages serves them perfectly as-is. Google Sheets only handles the optional cloud sync, which is a simple key-value read/write.

## Part 1: GitHub Pages (Hosting)

### Step 1: Create a GitHub repository

1. Go to [github.com/new](https://github.com/new)
2. Repository name: `study-apps` (or whatever you like)
3. Public (required for free GitHub Pages)
4. Check **"Add a README file"**
5. Click **Create repository**

### Step 2: Upload your files

1. In the repo, click **Add file → Upload files**
2. Upload these files:
   - `index.html` (landing page)
   - `gke-study-app.html`
   - `otca-study-app.html`
3. Click **Commit changes**

### Step 3: Enable GitHub Pages

1. Go to **Settings → Pages** (left sidebar)
2. Source: **Deploy from a branch**
3. Branch: **main** / **/ (root)**
4. Click **Save**
5. Wait ~60 seconds, then refresh — you'll see a URL like:
   `https://YOUR-USERNAME.github.io/study-apps/`

### Step 4: Test

- `https://YOUR-USERNAME.github.io/study-apps/` → landing page
- `https://YOUR-USERNAME.github.io/study-apps/gke-study-app.html` → GKE app
- `https://YOUR-USERNAME.github.io/study-apps/otca-study-app.html` → OTCA app

Everything works immediately — quiz, flashcards, localStorage persistence. No backend needed.

### Updating a course

1. In the GitHub repo, click the file → pencil icon (edit) or **Add file → Upload files** (replace)
2. Commit the change
3. GitHub Pages auto-deploys in ~30 seconds

---

## Part 2: Google Sheets Sync (Optional)

This adds cross-device progress sync. Your quiz answers, flashcard progress, and scores sync to a Google Sheet so you can continue on another device.

### Step 1: Create a Google Sheet

1. Go to [sheets.google.com](https://sheets.google.com) → **Blank spreadsheet**
2. Name it **"Crash Course Sync"**
3. In Row 1, type these headers:
   - A1: `key`
   - B1: `data`
   - C1: `savedAt`

### Step 2: Add the Apps Script

1. In your Sheet → **Extensions → Apps Script**
2. Click on `Code.gs` (already exists), delete everything, paste:

```javascript
var SHEET_NAME = 'Sheet1';

function doGet(e) {
  return handleSync(e, 'load');
}

function doPost(e) {
  return handleSync(e, 'save');
}

function handleSync(e, action) {
  try {
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
    if (!sheet) throw new Error('Sheet not found');
    var key = (e.parameter && e.parameter.key) || 'default';

    if (action === 'save') {
      var payload = e.postData ? JSON.parse(e.postData.contents) : {};
      var data = JSON.stringify(payload.data || {});
      var savedAt = payload.savedAt || new Date().toISOString();
      var rows = sheet.getDataRange().getValues();
      var rowIdx = -1;
      for (var i = 1; i < rows.length; i++) {
        if (rows[i][0] === key) { rowIdx = i + 1; break; }
      }
      if (rowIdx > 0) {
        sheet.getRange(rowIdx, 2).setValue(data);
        sheet.getRange(rowIdx, 3).setValue(savedAt);
      } else {
        sheet.appendRow([key, data, savedAt]);
      }
      return ContentService.createTextOutput(
        JSON.stringify({ ok: true, savedAt: savedAt })
      ).setMimeType(ContentService.MimeType.JSON);
    } else {
      var rows = sheet.getDataRange().getValues();
      for (var i = 1; i < rows.length; i++) {
        if (rows[i][0] === key) {
          return ContentService.createTextOutput(
            JSON.stringify({ ok: true, data: JSON.parse(rows[i][1] || '{}'), savedAt: rows[i][2] || null })
          ).setMimeType(ContentService.MimeType.JSON);
        }
      }
      return ContentService.createTextOutput(
        JSON.stringify({ ok: true, data: null, savedAt: null })
      ).setMimeType(ContentService.MimeType.JSON);
    }
  } catch (err) {
    return ContentService.createTextOutput(
      JSON.stringify({ ok: false, error: err.message })
    ).setMimeType(ContentService.MimeType.JSON);
  }
}
```

That's it — one file, ~45 lines. No `loader.html`, no `DriveApp`, no `HtmlService`.

### Step 3: Deploy

1. Click **Deploy → New deployment**
2. Gear icon → **Web app**
3. Settings:
   - Description: `Crash Course Sync`
   - Execute as: **Me**
   - Who has access: **Anyone**
4. Click **Deploy**
5. Copy the **Web App URL**

### Step 4: Connect sync in the app

1. Open your study app (on GitHub Pages or locally)
2. Go to **Settings → Cloud Sync**
3. Paste the Web App URL
4. (Optional) Enter your name in the **"Your name"** field — this keeps your data in a separate row if you share the sync sheet with others
5. Click **Save**
6. You should see "Connected" status

Progress now syncs across every device where you paste that same URL and name.

**Sharing with others:** If you share the GitHub Pages link and sync URL with teammates, each person just enters a different name (e.g. "barry", "sarah"). Their data syncs to separate rows in the Sheet — no interference. If the name field is left blank, everyone shares the same row (last write wins).

---

## How sync works

| Event | What happens |
|---|---|
| Answer a quiz, flip a card, etc. | Saved to localStorage instantly |
| 2 seconds of inactivity | Background sync pushes to Google Sheet |
| Open on another device | Fetches from Sheet, compares timestamps, uses newer data |
| Go offline | App works with localStorage, queues changes |
| Come back online | Auto-flushes pending changes to Sheet |
| Multiple courses | Each uses its own `STORAGE_KEY` as the row key — one Sheet handles all |

## Adding a new course

1. Build a new HTML study app (from `crash-course-TEMPLATE.html`)
2. Upload it to the GitHub repo
3. GitHub Pages serves it automatically
4. Sync works with the same Apps Script — each app uses its own `STORAGE_KEY`

## Troubleshooting

| Problem | Fix |
|---|---|
| GitHub Pages 404 | Check the file name matches exactly (case-sensitive). Ensure Pages is enabled in Settings → Pages. |
| Apps Script "Sheet not found" | Rename your sheet tab to `Sheet1`, or change `SHEET_NAME` in the script. |
| Sync not working | Check browser console (F12) for CORS errors. Ensure the Apps Script is deployed as "Anyone" can access. |
| Sync URL not saving | Make sure you click "Save" in the Cloud Sync settings. The URL persists in localStorage. |
| Changes to Code.gs not taking effect | Create a **new version**: Deploy → Manage deployments → pencil → Version: New version → Deploy. |
| Old data on new device | The app compares timestamps and uses the newer data. If you want to force a fresh pull, clear localStorage. |
