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

Open the deployed page in Chrome on Android, tap the three-dot menu, then **Install app** (or **Add to Home screen**). It opens full-screen like a regular app. The service worker loads the newest version whenever you're online, so updates appear after you push them.

If you test locally, give each project its own port (for example `python3 -m http.server 8001`) and add that origin under Authorized JavaScript origins. Service workers are tied to the address, and a port previously used by another app can keep serving that app.

## First sign-in

Google will show an **"unverified app"** warning because the app is in Testing mode. Click **Advanced > Go to Mail Sweeper (unsafe)**, then grant access. This is expected for your own personal project.

## Using it

1. **Connect Gmail** and approve access.
2. **Scan Gmail.** Progress is saved as it goes, so you can **Pause** or close the tab and resume later with **Resume scan**.
3. **Sync with Gmail** catches the app up after you delete, label or receive mail directly in Gmail. It asks Gmail what changed since the last sync, so it takes seconds instead of a full rescan. If it's been too long (Gmail keeps change history for roughly a week), it falls back to a quicker full check that only downloads details for new messages. Run it before you review or label anything.
4. **Explore** shows where your storage is going: space by type and age, **Top senders** (the 25 senders using the most space), and biggest messages.

The access token lasts about an hour. If a scan or sync pauses and the header shows "Reconnect needed," tap **Reconnect & resume** to carry on where you left off.

## Troubleshooting

| Problem | Likely cause |
| --- | --- |
| `Error 400: origin_mismatch` | The page's origin isn't in Authorized JavaScript origins. Check the scheme, host, and port exactly. |
| `Error 403: access_denied` | Your Gmail address isn't listed under Test users. |
| "Google sign-in is still loading" | The Google script hasn't loaded yet. Wait a moment and click again. |
| Scan stops with HTTP 403 | Gmail API isn't enabled in this project, or the `gmail.modify` scope wasn't granted. |
| Changes to Google settings don't take effect | They can take a few minutes to propagate. Wait and retry. |
