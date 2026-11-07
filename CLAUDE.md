# CLAUDE.md

Guidance for Claude when working in this repo.

## What this is

**Mail Sweeper**: a browser-only PWA that helps Jennifer reclaim Google storage by cleaning up a very large Gmail archive (about 63,500 messages, 5.6 GB). It scans message metadata into IndexedDB, lets her explore where the space is, and lets her review and trash messages in bulk. There is no backend.

- Deployed on GitHub Pages: https://jenfly.github.io/gmail-cleanup/ (installed on her Android phone as a home-screen app).
- **She uses it on her phone.** Everything is mobile-first.
- Files: `index.html` (the whole app: inline CSS and JS), `README.md` (setup and troubleshooting), `manifest.webmanifest`, `sw.js` (network-first service worker), `icon-192.png`, `icon-512.png`.
- No build step, no package.json, no dependencies. Keep it that way unless asked. The only external script is Google's sign-in library.

## Safety rules (do not break these)

- **OAuth scope is only `gmail.modify`.** It allows reading, labelling, inserting and trashing, but not permanent deletion. Do not add the broad `mail.google.com` scope or any permanent-delete call. Emptying Trash is deliberately left to Jennifer in Gmail.
- Starred messages and anything already in Trash or Spam are never offered for trashing, and trash actions re-check STARRED and TRASH against Gmail just before acting.
- Destructive steps need an explicit confirmation, and trashing must stay undoable (the `trashLog` and "Undo last trash").
- **Never require a full rescan.** The first scan took about four hours because Gmail throttles hard (roughly 4 messages/sec effective). Use Sync (Gmail history API) or targeted fetches instead.
- Keep the local cache in step with Gmail as each step succeeds, not only at the end, so a failure partway leaves the cache matching what Gmail actually did.
- Never put other people's email addresses (for example from her screenshots) into code, tests or docs.

## Architecture

Single file, one inline `<script>`, organised in sections marked by comments: Auth, IndexedDB, Scan, Sync with Gmail, Explore + Review.

**Data model.** IndexedDB database `mailsweeper`, v1:
- Store `msgs`, keyPath `id`, one row per message: `{id, from, date, size, labels, bad?}`. `from` is the lower-cased sender address (`parseSender`), `date` is `internalDate` in ms, `size` is `sizeEstimate` in bytes, `labels` are Gmail label ids. `bad` marks messages Gmail refuses to return (shown as `(unreadable)`).
- Store `meta` (key/value): `scan` (page token for resuming), `history` (`{id, at}` for Sync), `trashLog` (last 10 trash runs, for Undo).
- Subjects are **not** cached in IndexedDB. They are fetched lazily for the visible Review page and kept in the in-memory `subjectCache`.

**Tabs.** Data (connect, scan, Sync), Explore (space by type/age/size, top senders, biggest messages; tapping a row opens Review with filters applied via `applyExploreFilter`), Review (collapsible filters, paginated list of 25, selection, action bar with Trash and a ⋯ menu).

**Useful helpers**
- `api(path, opts)`: Gmail REST call with pacing, adaptive rate, retry/backoff on 429/5xx/network errors. Errors carry `.status` and `.path`; a 401 throws `Error('expired')` (handle with `expireForAction()`). It adds a JSON `Content-Type` whenever a body is present and uses the base URL `.../gmail/v1/users/me`, so a different base (for example the upload endpoint) needs its own fetch.
- `batchLabel(ids, body, onBatch)`: `batchModify` in chunks of 1000, calling `onBatch(part)` after each accepted chunk.
- `bulkTrash(ids, undo, onDone)`: trash or restore, with per-message fallback; reports finished ids through `onDone`.
- `cacheMarker(label, add)`: returns an async function that adds or removes a label on the cached rows for given ids and saves them.
- `listIds(labelId)`: all message ids carrying a label.
- `putMany`, `deleteMany`, `getMeta`, `setMeta`, `run`: IndexedDB helpers.
- Review state: `reviewRows` (all cached rows), `matchRows` (rows matching the filters), `selected` (a Set of chosen ids, empty by default), `selectedRows()`, `recomputeMatches(reset)`, `renderReviewPage()`, `refreshReview()`.
- `say(text, alert)`: the status line. `keepAwake(on)`: screen wake lock during long operations.

## UI conventions

- Mobile-first and plain. CSS variables `--bg`, `--ink`, `--muted`, `--line`, `--accent`.
- 48px tap targets, 16px inputs (prevents iOS zoom), `env(safe-area-inset-*)` padding, no tables.
- Build DOM with `textContent` and `createElement` (not `innerHTML`) for anything that includes message data.
- Long-running actions need progress feedback and a way to stop.

## Testing

There is no test suite in the repo. What has worked:
- Syntax check: extract the inline script and run it through `new Function(script)`.
- Logic and DOM: `jsdom` (npm) loading `index.html` with `runScripts: 'dangerously'`. In `beforeParse`, provide a fake `indexedDB` (Map-backed), `CSS.escape` and a stubbed `confirm`. Replace `api` with a fake Gmail that keeps server-side state and can fail on the Nth call. `api`, `putMany` and other function declarations can be reassigned on `window`; `const` helpers such as `getMeta` need `window.eval(...)`.
- Visual: headless Chromium (Playwright) at 390x844, serving the folder with `python3 -m http.server`, with service workers blocked.
- Real Gmail behaviour can only be verified by Jennifer trying it. Always say plainly what was and was not tested.

## Working on this repo

- Locally, serve the project on its own port. A service worker from another app on the same port once hijacked the page. After deploying, hard refresh (the service worker is network-first, but the first load installs it).
- Changes have been delivered as numbered patches that Jennifer applies herself with `git am`. Base each on a fresh clone of `main`, export with `git format-patch -1 --start-number N`, and check the patch applies cleanly on another fresh clone. Check `git log` for the latest number, and give the patch file to her explicitly along with a clear "apply this patch" instruction.
- Keep changes small and focused; one concern per patch where practical.
- Keep explanations short. A plain list of what changed is enough.

## Parked ideas

- Per-sender or per-category expiry rules: judged the wrong first tool, since size and age are the useful axes. Revisit only if she asks.
- Emptying Trash from inside the app: would need the broad scope. Only if she asks, as a clearly marked separate step.
- Dropbox integration: decided against. Attachments are downloaded to her phone and uploaded to Dropbox by hand (Drive and Photos share Gmail's storage pool, so saving there frees nothing).
