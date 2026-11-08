# CLAUDE.md

Guidance for Claude when working in this repo.

## What this is

**Mail Sweeper**: a browser-only PWA that helps Jennifer reclaim Google storage by cleaning up a very large Gmail archive (about 63,500 messages, 5.6 GB). It scans message metadata into IndexedDB, lets her explore where the space is, and lets her review and trash messages in bulk. There is no backend.

- Deployed on GitHub Pages: https://jenfly.github.io/gmail-cleanup/ (installed on her Android phone as a home-screen app).
- **She uses it on her phone.** Everything is mobile-first.
- A few friends use it too, added as Test users on her existing OAuth client (Testing mode caps that list at 100). They each see the same unverified-app warning she does; the in-app help guide explains it.
- Files: `index.html` (the whole app: inline CSS and JS), `README.md` (setup and troubleshooting), `manifest.webmanifest`, `sw.js` (network-first service worker), `icon-192.png`, `icon-512.png`.
- No build step, no package.json, no dependencies. Keep it that way unless asked. The only external script is Google's sign-in library.

## Safety rules (do not break these)

- **OAuth scope is only `gmail.modify`.** It allows reading, labelling, inserting and trashing, but not permanent deletion. Do not add the broad `mail.google.com` scope or any permanent-delete call. Emptying Trash is deliberately left to Jennifer in Gmail.
- Starred messages and anything already in Trash or Spam are never offered for trashing, and trash actions re-check STARRED and TRASH against Gmail just before acting.
- Destructive steps need an explicit confirmation, and trashing must stay undoable (the `trashLog` and "Undo last trash").
- **Never require a full rescan.** The first scan took about four hours because Gmail throttles hard (roughly 4 messages/sec effective). Use Sync (Gmail history API) or targeted fetches instead.
- Keep the local cache in step with Gmail as each step succeeds, not only at the end, so a failure partway leaves the cache matching what Gmail actually did.
- Never put other people's email addresses (for example from her screenshots) into code, tests or docs.
- **A new or less-tested code path inside an existing bulk action must stay reachable only from a single-message/explicit-selection flow, never from the batch function itself** — confirmed with her (2026-11-07) on the strip-attachments "wrap for alternative/related" fallback: a confirm dialog or an available undo doesn't substitute for her actually looking at a result when something is running across many messages at once. Report back a two-step flow (preview/insert a check copy, then a separate manual "trash it now") rather than one combined automatic action.
- **Restore (Copy to another device) is replace-only, atomic, and same-account.** It validates the whole file before touching IndexedDB, writes `msgs`/`meta` in one transaction so a failure leaves the old cache untouched, and never carries `stripLog`, `trashLog` or `stripRunLog` between devices — those describe real operations on a specific device's Gmail session. Restore never writes anything to Gmail. Export and Restore both run the normal Sync (read-only toward Gmail), and only in the "Sync with Gmail" state (history exists, no unfinished scan). Export checks the account first, then syncs, and writes no file if that sync didn't finish. Restore syncs after writing the cache and keeps the restored cache even if that follow-up sync doesn't finish.

## Architecture

Single file, one inline `<script>`, organised in sections marked by comments: Auth, IndexedDB, Scan, Sync with Gmail, Explore + Review.

**Data model.** IndexedDB database `mailsweeper`, v1:
- Store `msgs`, keyPath `id`, one row per message: `{id, from, date, size, labels, bad?}`. `from` is the lower-cased sender address (`parseSender`), `date` is `internalDate` in ms, `size` is `sizeEstimate` in bytes, `labels` are Gmail label ids. `bad` marks messages Gmail refuses to return (shown as `(unreadable)`).
- Store `meta` (key/value): `scan` (page token for resuming), `history` (`{id, at}` for Sync), `account` (lower-cased signed-in email, stamped by `scan()` and by export/restore — used to refuse exporting or restoring a saved scan across different Google accounts), `trashLog` (last 10 trash runs, for Undo), `attachIds` (cached ids with attachments for the Has-attachments filter, refetched only after a Sync), `stripLog` (crash safety: a stripped copy that's been inserted and verified but not yet trashed, so a resumed run never double-inserts), `stripRunLog` (last 10 strip runs, for Undo last strip), `noticeDismissed` (the Data-tab "before you start" notice was dismissed), `stripExplainerSeen` (the strip-attachments first-use explainer has been shown and acknowledged), `guideSeen` (the help guide has been closed at least once, so it no longer opens by itself on launch).
- A "Copy to another device" export is a JSON file: the `msgs` store as-is (including `bad` rows) plus `account`, `history` and `scan` from `meta`. It deliberately excludes `trashLog`, `stripLog`, `stripRunLog`, `attachIds` and the onboarding flags (`noticeDismissed`, `stripExplainerSeen`, `guideSeen`) — those describe state local to one device's Gmail session, not the scan itself.
- Subjects are **not** cached in IndexedDB. They are fetched lazily for the visible Review page and kept in the in-memory `subjectCache`.

**Tabs.** Data (connect, scan, Sync), Explore (space by type/age/size, top senders, biggest messages; tapping a row opens Review with filters applied via `applyExploreFilter`), Review (collapsible filters, paginated list of 25, selection, action bar with Trash and a ⋯ menu — Strip attachments, a label-and-apply row, Undo trash / Undo strip, and a collapsible Advanced section holding Inspect message; see "Strip attachments" below).

**Useful helpers**
- `api(path, opts)`: Gmail REST call with pacing, adaptive rate, retry/backoff on 429/5xx/network errors. Errors carry `.status` and `.path`; a 401 throws `Error('expired')` (handle with `expireForAction()`). It adds a JSON `Content-Type` whenever a body is present and uses the base URL `.../gmail/v1/users/me`, so a different base (for example the upload endpoint) needs its own fetch.
- `batchLabel(ids, body, onBatch)`: `batchModify` in chunks of 1000, calling `onBatch(part)` after each accepted chunk.
- `bulkTrash(ids, undo, onDone)`: trash or restore, with per-message fallback; reports finished ids through `onDone`.
- `cacheMarker(label, add)`: returns an async function that adds or removes a label on the cached rows for given ids and saves them.
- `listIds(labelId)`: all message ids carrying a label.
- `putMany`, `deleteMany`, `getMeta`, `setMeta`, `run`: IndexedDB helpers. `runMulti(stores, mode, fn)`: like `run`, but for one transaction spanning several stores at once — used only by restore, so `msgs` and `meta` are replaced atomically.
- `sync()` now returns whether it completed: `true` only when the history sync finished or the full-check fallback (`scan({fresh:true})`) ran all the way through; pausing, an expired session or any error returns `false`. `scan()` itself now returns the same kind of completion boolean. Neither changes what the Data-tab buttons show.
- Review state: `reviewRows` (all cached rows), `matchRows` (rows matching the filters), `selected` (a Set of chosen ids, empty by default), `selectedRows()`, `recomputeMatches(reset)`, `renderReviewPage()`, `refreshReview()`.
- `say(text, alert)`: the status line. `keepAwake(on)`: screen wake lock during long operations.
- `exportCache()` / `handleRestoreFile(file)`: the "Copy to another device" export and restore (see the Data model and safety rules above). `getAccountEmail()`, `validateCachePayload(payload)`, `restoreConfirmText(payload, existingCount)` support them.

**Strip attachments.** For selected messages with a large attachment (`STRIP_MIN_BYTES`, 50KB and up): zips the attachments to her device, inserts a copy of the message with those attachments removed (same headers/thread/labels, a short note added, labelled `Mail Sweeper review`), then trashes the original. Inline images and attachments under the threshold are left in place; Sent messages are included.
- `analyzeMime(rawBin)`: parses nested MIME recursively and classifies each part (remove/keep-small/inline/unsupported — signed, encrypted, attached-email).
- `buildStrippedRaw(s, analysis, opts)`: rebuilds the message with target parts excised, byte-for-byte otherwise (`rebuildPart`/`hasRemovalBeneath`/`STRIP_DROP`). `opts.allowWrap` additionally allows wrapping an `alternative`/`related` root in a new `multipart/mixed` container — reachable only from the single-message preview's "Build wrapped copy to check…" button, **never** from the batch pipeline (see the safety rule above).
- `stripOne(m)`: the per-message pipeline (download zip → insert → verify → trash original); used by the batch run `stripSelected()` / undone by `undoStrip()` via `stripRunLog`.
- `previewStripFor(m)` / `previewStrip()`: read-only inspection for exactly one message (⋯ > Advanced > Inspect message). `copyStructure` / `buildStructureOutline` copy a plain-text MIME outline (content types, sizes, removal flags — no subject, sender, body or filenames) for sharing a skipped message's shape without exposing real content.

**Help and onboarding.** A `#helpBtn` in the header opens `#guideOverlay`, a full-screen scrolling panel (no fifth tab). It starts with a `#guideQuick` "Quick start" card (five short steps: pick where to scan first, sign in, scan — mentioning that the scan can then be copied to another device, explore/review, empty Trash in Gmail, plus a "Got it" button), then a "More detail" list of eleven collapsible `<details>` rows covering: picking where to scan first (`#guideStart`), installing on the phone (`#guideInstall`; its Install button shows only while the browser holds a `beforeinstallprompt`, otherwise manual steps, and it tells iPhone users to install before the first scan because home-screen storage can be separate from Safari's), privacy, first sign-in/Testing-mode warnings, Scan, Sync, Copy to another device (`#guideCopy`, between Sync and Explore), Explore, Review, Strip attachments, and emptying Trash. Only one row is open at a time. `openGuide(anchorId)` shows the guide with every row collapsed, then expands and scrolls to the one named (e.g. `guideStrip`); `closeGuide()` hides it and sets `meta.guideSeen`. It's a CSS overlay, not a tab switch, so whatever was happening underneath (a running scan, Review's filters/selection) is untouched. On launch the guide opens by itself until `guideSeen` is set (closing it any way, X or "Got it", counts), and after that only the Help button opens it; the guide's wording is the source of truth, so keep the quick-start steps in step with the sections below them. The Data tab's `#startNotice` repeats the first guide section above the Scan button; `updateDataAction()` shows it only when there's no scan progress (`meta.scan.pageToken`) and no cached messages, and hides it permanently once "Got it" is tapped (`meta.noticeDismissed`). `startStrip()` gates the Strip attachments menu row: the first time (`!meta.stripExplainerSeen`), it shows `#stripExplainer` — a short modal — before calling `stripSelected()`; continuing or ticking "Don't show this again" sets the seen flag, a plain cancel does not (so it reappears), and the modal's "See the full guide section" link opens `#guideStrip` without marking it seen. `stripSelected()` itself is unchanged by any of this — see the batch-safety rule above, which also applies to onboarding UI: it must never short-circuit or auto-confirm the batch pipeline's own checks.

## UI conventions

- Mobile-first and plain. CSS variables `--bg`, `--ink`, `--muted`, `--line`, `--accent`.
- 48px tap targets, 16px inputs (prevents iOS zoom), `env(safe-area-inset-*)` padding, no tables.
- Build DOM with `textContent` and `createElement` (not `innerHTML`) for anything that includes message data. Static icon markup is fine as inline `<svg>` since it holds no message data.
- Icons are inline SVG, 20px, `stroke="currentColor"`, outline style (Tabler-style paths), `aria-hidden="true"`. No icon font, no external library.
- Long-running actions need progress feedback and a way to stop.

## Testing

There is no test suite in the repo. What has worked:
- Syntax check: extract the inline script and run it through `new Function(script)`.
- Logic and DOM: `jsdom` (npm) loading `index.html` with `runScripts: 'dangerously'`. In `beforeParse`, provide a fake `indexedDB` — a Map-backed store per object store, with every request resolved via `setTimeout(...,0)` and each transaction's `oncomplete` fired on a slightly longer timeout so it settles after that turn's requests — plus `CSS.escape`, a stubbed `confirm`, and `Element.prototype.scrollIntoView`. Replace `api` with a fake Gmail that keeps server-side state and can fail on the Nth call. `api`, `updateDataAction`, `startStrip` and other top-level `function` declarations can be called as `window.foo()`; top-level `const`/`let` helpers (`getMeta`, `setMeta`, `putMany`, `deleteMany`, ...) don't attach to `window` and need `window.eval('getMeta("key")')` instead (this is the JS spec, not a jsdom quirk — top-level `function` declarations in a classic script become global-object properties, lexical `const`/`let` don't).
- The fake `indexedDB`'s `transaction()` must accept an array of store names and let `objectStore()` pick among them, to exercise `runMulti`'s atomic `['msgs','meta']` restore write — and a request that fails inside a transaction must abort that transaction (fire `onabort`, not `oncomplete`) rather than just rejecting silently, or a hung promise can make the whole test process exit early with no error. Scope any simulated-failure flag narrowly (e.g. only a multi-store transaction) — a fake that aborts every transaction touching a given store will also break ordinary reads like `countMsgs()` that have no `onabort` handler. Stub `window.URL.createObjectURL`/`revokeObjectURL` (jsdom doesn't implement them) to capture the exported `Blob` for assertions instead of relying on the `<a download>` click. A `File`/`Blob`-like fake with a `.text()` method is enough for restore; jsdom's own `Blob.prototype.text` also works.
- Visual: headless Chromium (Playwright) at 390x844, serving the folder with `python3 -m http.server`, with service workers blocked. Block `https://accounts.google.com/**` via `page.route` *before navigating*, or the real GSI script can reach the network and overwrite a stubbed `window.google`.
- Gotcha: Chromium does not apply `display:none` to a bare `<svg>` via the HTML `hidden` attribute (jsdom won't catch this — it only showed up in a real Playwright render). Toggle `style.display` instead of `.hidden` when showing/hiding a raw SVG icon.
- Real Gmail behaviour can only be verified by Jennifer trying it. Always say plainly what was and was not tested.

## Working on this repo

- Locally, serve the project on its own port. A service worker from another app on the same port once hijacked the page. After deploying, hard refresh (the service worker is network-first, but the first load installs it).
- In chat sessions (no repo access), changes are delivered as numbered patches that Jennifer applies with `git am`: base each on a fresh clone of `main`, export with `git format-patch -1 --start-number N`, check it applies cleanly on another fresh clone, and give her the file with a clear "apply this patch" instruction. Check `git log` for the latest number (patches 0001 to 0015 are applied). In Claude Code sessions, patches are not needed: commit directly in the repo.
- Keep changes small and focused; one concern per patch where practical.
- Keep explanations short. A plain list of what changed is enough.

## Parked ideas

- Per-sender or per-category expiry rules: judged the wrong first tool, since size and age are the useful axes. Revisit only if she asks.
- Emptying Trash from inside the app: would need the broad scope. Only if she asks, as a clearly marked separate step.
- Dropbox integration: decided against. Attachments are downloaded to her phone and uploaded to Dropbox by hand (Drive and Photos share Gmail's storage pool, so saving there frees nothing).
