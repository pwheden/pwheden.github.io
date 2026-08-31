# Newsletter signup — Google Sheet setup

The signup form in the site footer sends each email address to a Google Apps
Script web app, which appends it as a row in a Google Sheet you own. This file
walks through the one-time setup (~10 minutes). Everything is free.

## 1. Create the Google Sheet

1. Go to https://sheets.google.com and create a new spreadsheet.
2. Name it e.g. **Nyhetsbrev – prenumeranter**.
3. In row 1, type headers: `E-post` in A1, `Datum` in B1 (optional but nice).

## 2. Add the Apps Script

1. In the sheet, open **Extensions → Apps Script**.
2. Delete any code in the editor and paste in the script below.
3. Click the save icon (name the project e.g. "Nyhetsbrev").

```javascript
function doPost(e) {
  const lock = LockService.getScriptLock();
  lock.tryLock(10000);
  try {
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
    const email = (e.parameter.email || '').trim().toLowerCase();

    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      return respond('invalid');
    }

    const lastRow = sheet.getLastRow();
    if (lastRow > 0) {
      const existing = sheet.getRange(1, 1, lastRow, 1).getValues()
        .some(row => String(row[0]).trim().toLowerCase() === email);
      if (existing) return respond('duplicate');
    }

    sheet.appendRow([email, new Date()]);
    return respond('ok');
  } catch (err) {
    return respond('error');
  } finally {
    lock.releaseLock();
  }
}

function respond(result) {
  return ContentService.createTextOutput(JSON.stringify({ result: result }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## 3. Deploy it as a web app

1. Click **Deploy → New deployment** (top right).
2. Click the gear icon next to "Select type" and choose **Web app**.
3. Set:
   - **Execute as:** Me (pwheden@gmail.com)
   - **Who has access:** Anyone  ← required, otherwise the form can't reach it
4. Click **Deploy**, then **Authorize access** and approve with your Google
   account (Google shows a warning because it's your own unverified script —
   click "Advanced" → "Go to … (unsafe)" → Allow).
5. Copy the **Web app URL** (looks like
   `https://script.google.com/macros/s/AKfy.../exec`).

## 4. Connect the website

In `index.html`, find this line in the `<script>` section:

```javascript
const NEWSLETTER_ENDPOINT = 'PASTE_YOUR_APPS_SCRIPT_URL_HERE';
```

Replace the placeholder with the web app URL from step 3, commit, and push.
(Or just paste the URL into a Claude Code chat and ask it to wire it up.)

## Notes

- **Sending a newsletter:** copy the email column from the sheet and paste
  into Gmail's **BCC** field (never To/CC — recipients must not see each
  other's addresses). Gmail allows ~500 recipients per day.
- **Unsubscribes:** the privacy policy tells subscribers to email you to
  unsubscribe — just delete their row in the sheet. Include a short
  "Vill du avsluta prenumerationen? Svara på detta mejl." line in every
  newsletter you send.
- **Updating the script later:** after editing the script you must click
  **Deploy → Manage deployments → ✏️ Edit → Version: New version → Deploy**,
  otherwise the old code keeps running. The URL stays the same.
- Duplicate signups are detected by the script; the visitor sees
  "Du prenumererar redan".
