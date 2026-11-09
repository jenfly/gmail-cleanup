# Mail Sweeper

A browser-only tool for finding what's taking up space in your Gmail archive. It scans message metadata (sender, date, size, labels), caches it in your browser's IndexedDB, and reports storage by sender. Nothing is sent anywhere except to Google's Gmail API.

## One-time Google setup

You need your own OAuth client ID so the page can sign in to your Gmail.

### 1. Create a project
1. Go to <https://console.cloud.google.com>.
2. Click the project picker at the top, then **New project**. Name it something like `mail-sweeper` and create it.
3. Make sure the new project is selected.

### 2. Enable the Gmail API
1. Open **APIs & Services > Library**.
2. Search for **Gmail API**, open it, and click **Enable**.

### 3. Configure the consent screen
1. Open **APIs & Services > OAuth consent screen** (shown as **Google Auth Platform** in newer consoles) and click **Get started** if prompted.
2. Enter an app name (e.g. `Mail Sweeper`) and your email as the support contact.
3. Set the audience to **External**.
4. Add your email as the developer contact, then finish.
5. Under **Audience**, confirm the publishing status is **Testing**, and add your own Gmail address under **Test users**.
6. Under **Data Access** (or **Scopes**), add this scope:
   `https://www.googleapis.com/auth/gmail.modify`

Leave the app in Testing mode. You don't need to publish it or go through Google's verification, since you're the only user.

### 4. Create the OAuth client ID
1. Open **APIs & Services > Credentials** (or **Clients** in the newer console).
2. Click **Create credentials > OAuth client ID**.
3. Application type: **Web application**.
4. Under **Authorized JavaScript origins**, add:
   - `http://localhost:8000` (for testing on your computer)
   - `https://jenfly.github.io` (for the deployed page)
5. Leave **Authorized redirect URIs** empty. The sign-in popup doesn't use them.
6. Click **Create** and copy the **Client ID** (it ends in `.apps.googleusercontent.com`).

Origins are just the scheme and host. Don't include the repo path or a trailing slash.

### 5. Add the client ID to the app
Open `index.html` and replace the placeholder near the top of the script:

```js
const CLIENT_ID = 'YOUR_CLIENT_ID.apps.googleusercontent.com';
```

The client ID isn't a secret, so it's fine to commit it to a public repo.

## Run it

**Locally:** from the folder containing `index.html`:

```
python3 -m http.server 8000
```

Then open <http://localhost:8000>.

**On GitHub Pages:** push `index.html` to a repo, enable Pages in the repo settings, and open the Pages URL.

## Install on your phone

**Android (Chrome):** open the deployed page, tap the three-dot menu, then **Install app** (or **Add to Home screen**).

**iPhone (Safari):** open the deployed page, tap the Share icon, then **Add to Home Screen**. (Must be Safari — Chrome and other browsers on iOS can't install home-screen apps.)

The in-app help guide has the same instructions under "Install on your phone", with an Install button where the browser supports it.

Either way it opens full-screen like a regular app. The service worker loads the newest version whenever you're online, so updates appear after you push them.

If you test locally, give each project its own port (for example `python3 -m http.server 8001`) and add that origin under Authorized JavaScript origins. Service workers are tied to the address, and a port previously used by another app can keep serving that app.

## First sign-in

Google will show an **"unverified app"** warning because the app is in Testing mode. Click **Advanced > Go to Mail Sweeper (unsafe)**, then grant access. This is expected for your own personal project.

## Sharing it with others

To let someone else use your copy of Mail Sweeper, add their Gmail address under **OAuth consent screen > Audience > Test users** (step 3 in setup, above). They sign in using your existing OAuth client — there's nothing else to set up on your end. They'll see the same "unverified app" warning you do; that's expected in Testing mode. Testing mode allows up to 100 test users.

Mail Sweeper has a built-in help guide (tap the **?** in the header) that walks a new user through all of this, including picking a device for the first scan and what the sign-in warning means.

## Using it

Tap the **?** in the header any time for the full in-app guide.

1. **Connect Gmail** and approve access.
2. **Scan Gmail.** The first scan can take hours, and what it finds is stored only on this device and browser. A notice above the Scan button covers this until you dismiss it (it also goes away on its own once a scan starts). Progress is saved as it goes, so you can **Pause** or close the tab and resume later with **Resume scan**. Once it's done, you can copy the scan to another device instead of scanning again there — see "Copying a scan to another device" below.
3. **Sync with Gmail** catches the app up after you delete, label or receive mail directly in Gmail. It asks Gmail what changed since the last sync, so it takes seconds instead of a full rescan. If it's been too long (Gmail keeps change history for roughly a week), it falls back to a quicker full check that only downloads details for new messages. Run it before you review or label anything.
4. **Explore** shows where your storage is going: space by type and age, **Top senders** (the 25 senders using the most space), and biggest messages. Tapping a row opens **Review** with that filter applied.
5. **Review** lists messages matching your filters (sender, size, age, Sent/Archived, Has attachments). Select the ones you want, then use the bar at the bottom:
   - **Trash** moves the selection to Gmail Trash. Starred and already-trashed messages are never included, even if selected.
   - The **⋯** menu can also **apply a label** to the selection, and **strip attachments from selected…** — for each message with a large attachment, it zips the attachment(s) to your device, replaces the message with an attachment-free copy (labelled `Mail Sweeper review` so it's easy to find), and trashes the original. Messages without a large attachment are skipped and left alone. The first time you tap it, a short explainer covers what it does before the usual confirmation; tick "Don't show this again" or continue through it once, and it won't reappear.
   - **Undo trash** and **Undo strip** reverse the most recent run of each.
   - Under **Advanced**, **Inspect message** shows what a strip would remove from exactly one selected message, without changing anything — useful to check a message first, or to see why a message was skipped in a run.

Emptying Gmail's Trash (to actually free the space) is a separate step you do in Gmail — this app never does it for you.

The access token lasts about an hour. If a scan or sync pauses and the header shows "Reconnect needed," tap **Reconnect & resume** to carry on where you left off.

## Copying a scan to another device

The scan lives only in the browser that ran it. To clean up from a second device (or back up the scan before clearing site data or uninstalling the app), use **Copy to another device** on the Data tab:

1. On the device with the scan, tap **Export saved scan**. This syncs with Gmail first, then downloads a compressed file (`mail-sweeper-cache-YYYY-MM-DD.json.gz`) to your Downloads folder. It lists sender addresses, dates and sizes for your whole mailbox, so keep it private.
2. Move that file to the other device (email it to yourself, a cloud drive, a USB stick — whatever's easiest).
3. On the other device, sign in with **the same Google account**, then tap **Restore from file** and pick the file. This replaces whatever scan was already saved there, then syncs with Gmail automatically. Restore also accepts the plain `.json` files made by earlier versions of the app. If that device's browser is too old to open the compressed file, unzip the `.gz` file on a computer first (7-Zip on Windows; Mac and Linux can open it natively) and restore the resulting `.json` file instead.

Each device then syncs independently from then on; a change made on one shows up on the other at its next Sync. Undo (trash and strip) only covers what was done on that specific device, since each copy has its own history. Restoring a file more than about a week old can trigger a slower full sync on the first run, same as reopening the app after a long gap.

Nothing is uploaded anywhere — the file is created on your device and goes only where you put it.

## Troubleshooting

| Problem | Likely cause |
| --- | --- |
| `Error 400: origin_mismatch` | The page's origin isn't in Authorized JavaScript origins. Check the scheme, host, and port exactly. |
| `Error 403: access_denied` | Your Gmail address isn't listed under Test users. |
| "Google sign-in is still loading" | The Google script hasn't loaded yet. Wait a moment and click again. |
| Scan stops with HTTP 403 | Gmail API isn't enabled in this project, or the `gmail.modify` scope wasn't granted. |
| Changes to Google settings don't take effect | They can take a few minutes to propagate. Wait and retry. |
