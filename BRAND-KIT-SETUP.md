# Brand Discovery Kit — delivery setup

The kit at `/brand-discovery-kit/` works with no setup: answers autosave to the
browser and **Download PDF** always works. Two optional integrations make
**Submit Brief** do something useful. Both are free.

Configure them at the top of the `// ── Delivery config ──` block in
`brand-discovery-kit/index.html`. Either works on its own.

| Constant | What it does | Required? |
| --- | --- | --- |
| `WEB3FORMS_KEY` | Emails each brief to you | Recommended |
| `SHEET_ENDPOINT` | Appends each brief as a row in a Google Sheet | Optional |

> **This file and the kit are public.** Both values above are safe to publish —
> the Web3Forms key only *accepts* submissions and can't read past ones, and the
> Apps Script URL is write-only. **Never** put a Google API key, Airtable token,
> or Notion secret in the page; those grant read and delete access to everything.

---

## 1. Email delivery (Web3Forms)

1. Go to <https://web3forms.com> and enter the inbox that should receive briefs.
2. They email you an **access key** (a UUID). No account or password needed.
3. Paste it into `brand-discovery-kit/index.html`:

   ```js
   const WEB3FORMS_KEY = 'your-access-key-here';
   ```

Use a **different key from the landing page's lead form** (`index.html`), so
client briefs don't land in the same pile as marketing leads.

### Known limits

- **Attachments are PRO-only.** Web3Forms' docs: *"Basic file uploads (up to 5MB,
  single file) require a PRO plan."* So the kit sends the full brief as **text in
  the email body** rather than attaching the generated PDF. Nothing is lost — the
  client still has the Download PDF button for their own copy.
- The monthly submission cap on the free tier isn't documented publicly; check
  <https://web3forms.com/pricing> if you expect volume.
- Spam protection: the kit sends the free `botcheck` honeypot field, matching the
  landing page form. If spam becomes a problem, Web3Forms offers zero-config
  hCaptcha on free.

---

## 2. Google Sheet (Apps Script)

Free, no row limits worth worrying about (~10M cells per spreadsheet), and no
account tier that expires or pauses.

### Steps

1. Create a new Google Sheet. Name it whatever you like.
2. **Extensions → Apps Script.** This creates a script bound to that sheet.
3. Delete the placeholder `myFunction` and paste in the code below.
4. **Deploy → New deployment.**
   - Click the gear next to "Select type" → **Web app**
   - *Description:* anything
   - *Execute as:* **Me**
   - *Who has access:* **Anyone** — note this is **not** "Anyone with a Google
     account". If you pick the wrong one, submissions silently fail.
5. Authorize when prompted. You'll see an "unverified app" warning because it's
   your own script — **Advanced → Go to (project name)**.
6. Copy the **Web app URL**. It ends in `/exec`.
7. Paste it into `brand-discovery-kit/index.html`:

   ```js
   const SHEET_ENDPOINT = 'https://script.google.com/macros/s/AKfy.../exec';
   ```

> **After any edit to the script**, you must deploy again —
> **Deploy → Manage deployments → edit (pencil) → Version: New version → Deploy.**
> Editing the code alone does nothing to the live URL.

### The script

```javascript
const SHEET_NAME = 'Briefs';

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    getSheet_().appendRow([
      new Date(),
      data.submitted_at || '',
      data.client_name || '',
      data.founders || '',
      data.email || '',
      data.phone || '',
      data.completion || '',
      data.brief || ''
    ]);
    return json_({ ok: true });
  } catch (err) {
    return json_({ ok: false, error: String(err) });
  }
}

function getSheet_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(SHEET_NAME) || ss.insertSheet(SHEET_NAME);
  if (sheet.getLastRow() === 0) {
    sheet.appendRow([
      'Received', 'Submitted at', 'Client', 'Founders',
      'Email', 'Phone', 'Completion', 'Full brief'
    ]);
    sheet.setFrozenRows(1);
  }
  return sheet;
}

function json_(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### Why the sheet write is "fire and forget"

Apps Script web apps don't return CORS headers a browser can read, so the kit
posts with `mode: 'no-cors'` and gets back an opaque response it cannot inspect.
Consequences worth knowing:

- The success toast reflects the **email** result only. A sheet write could fail
  silently.
- **Send one test brief after setup and confirm the row appears.** That's the only
  way to know it's wired up.
- The POST uses `Content-Type: text/plain` on purpose. That keeps it a "simple
  request" so the browser skips the CORS preflight, which Apps Script rejects.
  `doPost` reads the JSON out of `e.postData.contents`.

---

## Options that were considered and rejected

| Option | Why not |
| --- | --- |
| **Supabase** | Free tier is a real Postgres with a table editor, and its anon key is designed to be public with row-level security. But *"Free projects are paused after 1 week of inactivity"* — for a form used a few times a month, a client would hit a dead endpoint. |
| **Airtable / Notion** | Best table UI of the bunch, but their API tokens grant read **and delete** on the whole base. Exposed in public page source, anyone could drain or wipe your data. Only safe behind a relay like Zapier or Make. |
| **Formspree** | What the original file was written against. Works, but means another signup, and its free tier is tighter than Web3Forms — which this site already uses. |
