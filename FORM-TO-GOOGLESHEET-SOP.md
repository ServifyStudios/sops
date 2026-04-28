# SOP — Form Submissions → Google Sheets (via Google Apps Script)

**Owner:** Servify Studios  
**Applies to:** Any AI (Claude Code or other) building client websites with contact/application forms  
**Last updated:** April 2026

---

## Overview

Every client website has **one Google Sheet**. Each form on the website gets **one tab** in that sheet. All submissions from all forms go into the same spreadsheet, cleanly separated by tab.

A Google Apps Script deployed as a web app acts as the webhook — it receives POST requests from the form, formats the data, writes to the correct tab, and sends an email notification.

---

## Scenario A — First-Time Setup (New Client Project)

### Step 1 — Gather from the client/user before writing any code

Ask for these before doing anything:

| Item | Why needed |
|------|-----------|
| **Google account email** that owns the Sheet | Apps Script runs under this account |
| **Form fields** (labels + input types) | Determines columns in the sheet |
| **Who receives email notifications** | Email address(es) for `MailApp.sendEmail()` |
| **Notification subject line** | e.g. "New AJC Application Received" |
| **Client/project name** | Names the spreadsheet and script |

### Step 2 — Create the Google Sheet

1. Go to [sheets.google.com](https://sheets.google.com) → **New spreadsheet**
2. Name it: `[Client Name] — Website Forms`
3. Rename the default tab (Sheet1) to match the first form name, e.g. `Application Form`
4. Add the standard column headers in Row 1 (see **Column Headers** section below)

### Step 3 — Create the Apps Script

1. Inside the Google Sheet: **Extensions → Apps Script**
2. Delete any default code
3. Paste the **Master Script** (see below)
4. Update the `NOTIFY_EMAIL` constant at the top
5. Save (Ctrl+S / Cmd+S), name the project e.g. `[Client] Form Handler`

### Step 4 — Deploy as Web App

1. Click **Deploy → New deployment**
2. Click the gear icon next to "Select type" → choose **Web app**
3. Settings:
   - Description: `v1`
   - Execute as: **Me**
   - Who has access: **Anyone**
4. Click **Deploy** → Authorise when prompted (allow all permissions)
5. Copy the **Web App URL** — this is the webhook endpoint

### Step 5 — Wire up the form

Paste the webhook URL into the form's JavaScript `fetch()` call:

```javascript
fetch('PASTE_WEB_APP_URL_HERE', {
  method: 'POST',
  mode: 'no-cors',   // required — avoids CORS preflight error
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload)
});
```

> **Why `no-cors`?** Google Apps Script web apps don't return CORS headers. `no-cors` sends the request successfully but returns an opaque response (no status code readable in JS). The data still lands in the sheet. Handle UI feedback optimistically (show success toast after 1s timeout or on `fetch` resolve).

### Step 6 — Test

Send a test submission → verify:
- [ ] Row appears in the correct sheet tab
- [ ] Date column shows `DD Month YYYY` format (e.g. `28 April 2026`)
- [ ] Notification email arrives at the correct address(es)
- [ ] Form resets and shows success message

---

## Scenario B — New Form on an Existing Client Project

The Google Sheet already exists. No new spreadsheet needed.

### What the AI does automatically (no need to ask the client):

1. **Identifies the existing Apps Script** — ask the user for the existing webhook URL, or re-open the Apps Script from Extensions menu
2. **Creates a new tab** in the existing sheet named after the new form
3. **Adds headers** to the new tab (standard columns + any form-specific extras)
4. **Updates the Apps Script** — the master script handles all tabs dynamically, so no code change needed unless the form has custom fields
5. **Adds the email notification** for the new form inside the script's routing logic
6. **Re-deploys** with a new version (Deploy → Manage deployments → Edit → bump version)
7. **Wires up the new form HTML/JS** with the same webhook URL

> The webhook URL stays the same — the script routes by `form_name` field.

---

## Column Headers (Copy-Paste)

Paste this into **Row 1, starting at Column A** of each new tab:

```
Date	Form Name	Name	Email	Duration	Revenue	Challenge	Message
```

> These are tab-separated. In Google Sheets, paste into cell A1 → it will auto-split across columns.

**Column definitions:**

| Column | Description |
|--------|-------------|
| `Date` | Auto-set by script. Format: `28 April 2026` |
| `Form Name` | Auto-set by script from `form_name` field in payload |
| `Name` | Submitter's full name |
| `Email` | Submitter's email address |
| `Duration` | How long they've been coaching / in business |
| `Revenue` | Current revenue range |
| `Challenge` | Main challenge they're facing |
| `Message` | Free-text message / anything else |

> For forms with different fields, add extra columns to the right of `Message`. The script appends all extra fields at the end automatically.

---

## Master Apps Script (Copy-Paste)

Paste this into the Apps Script editor. Update the constants at the top only.

```javascript
// ── CONFIG ──────────────────────────────────────────────
var NOTIFY_EMAIL  = 'your@email.com';          // Who receives notifications
var NOTIFY_CC     = '';                         // CC address (leave blank if none)
var PROJECT_NAME  = 'AJC Coaching School';      // Used in email subject line
// ────────────────────────────────────────────────────────

function doPost(e) {
  try {
    var data = JSON.parse(e.postData.contents);
    var ss   = SpreadsheetApp.getActiveSpreadsheet();

    // ── Date: DD Month YYYY ──
    var months = [
      'January','February','March','April','May','June',
      'July','August','September','October','November','December'
    ];
    var now     = new Date();
    var dateStr = now.getDate() + ' ' + months[now.getMonth()] + ' ' + now.getFullYear();

    // ── Route to correct tab ──
    var tabName = data.form_name || 'Submissions';
    var sheet   = ss.getSheetByName(tabName);

    // Create tab if it doesn't exist yet
    if (!sheet) {
      sheet = ss.insertSheet(tabName);
    }

    // ── Add headers if sheet is empty ──
    if (sheet.getLastRow() === 0) {
      sheet.appendRow([
        'Date', 'Form Name', 'Name', 'Email',
        'Duration', 'Revenue', 'Challenge', 'Message'
      ]);
      sheet.getRange(1, 1, 1, sheet.getLastColumn())
        .setFontWeight('bold')
        .setBackground('#020E1D')
        .setFontColor('#D8C199');
    }

    // ── Append submission row ──
    sheet.appendRow([
      dateStr,
      data.form_name  || '',
      data.name       || '',
      data.email      || '',
      data.duration   || '',
      data.revenue    || '',
      data.challenge  || '',
      data.message    || ''
    ]);

    // ── Email notification ──
    var subject = '[' + PROJECT_NAME + '] New ' + tabName + ' Submission — ' + (data.name || 'Unknown');
    var body =
      'New form submission received.\n\n' +
      '━━━━━━━━━━━━━━━━━━━━━━\n' +
      'Form:      ' + tabName + '\n' +
      'Date:      ' + dateStr + '\n' +
      '━━━━━━━━━━━━━━━━━━━━━━\n\n' +
      'Name:      ' + (data.name      || '—') + '\n' +
      'Email:     ' + (data.email     || '—') + '\n' +
      'Duration:  ' + (data.duration  || '—') + '\n' +
      'Revenue:   ' + (data.revenue   || '—') + '\n' +
      'Challenge: ' + (data.challenge || '—') + '\n\n' +
      'Message:\n' + (data.message || '—') + '\n\n' +
      '━━━━━━━━━━━━━━━━━━━━━━\n' +
      'View sheet: ' + ss.getUrl();

    var mailOptions = { name: PROJECT_NAME };
    if (NOTIFY_CC) mailOptions.cc = NOTIFY_CC;

    MailApp.sendEmail(NOTIFY_EMAIL, subject, body, mailOptions);

    return ContentService
      .createTextOutput(JSON.stringify({ status: 'ok' }))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ status: 'error', message: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

---

## Form-Side JavaScript Pattern (Copy-Paste)

Standard fetch pattern for any form on the site:

```javascript
async function submitForm(formData) {
  const WEBHOOK = 'PASTE_WEB_APP_URL_HERE';

  const payload = {
    form_name:  'Application Form',   // Must match the sheet tab name exactly
    name:       formData.name,
    email:      formData.email,
    duration:   formData.duration,
    revenue:    formData.revenue,
    challenge:  formData.challenge,
    message:    formData.message
  };

  try {
    await fetch(WEBHOOK, {
      method:  'POST',
      mode:    'no-cors',
      headers: { 'Content-Type': 'application/json' },
      body:    JSON.stringify(payload)
    });
    // no-cors returns opaque response — treat resolve as success
    showToast('success', 'Received! We'll be in touch soon.');
    form.reset();
  } catch (err) {
    showToast('error', 'Something went wrong. Please email us directly.');
  }
}
```

---

## Re-deployment Checklist (when script is updated)

Every time the Apps Script code is changed, a **new version must be deployed**:

1. Extensions → Apps Script
2. Deploy → **Manage deployments**
3. Click the pencil (Edit) on the current deployment
4. Version → **"New version"**
5. Click **Deploy**

> The webhook URL does **not** change between versions. No update needed in the HTML.

---

## Security Notes

- The Apps Script runs under the owner's Google account — protect the webhook URL (treat it like an API key)
- Add a honeypot field to the HTML form (`display: none` input) and check for it in the script to block basic bot submissions:

```javascript
// In doPost(), before appending:
if (data.honeypot && data.honeypot !== '') {
  return ContentService.createTextOutput('ok');  // silently ignore bots
}
```

- Rate limiting: Google Apps Script has a 20,000 calls/day limit on free accounts (generous for most client sites)

---

## Quick Reference — Common Issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Data not appearing in sheet | Script not deployed as "Anyone" | Redeploy with correct access setting |
| Old code still running | Forgot to bump version on redeploy | Deploy → Manage → New version |
| No email arriving | Wrong email in `NOTIFY_EMAIL` | Update constant + redeploy |
| CORS error in console | Missing `mode: 'no-cors'` | Add to fetch options |
| All submissions land on Sheet1 | `form_name` field missing from payload | Add `form_name` key to JS payload object |
| Date shows as timestamp number | Sheet column formatted as number | Format column as Plain text in Sheets |
