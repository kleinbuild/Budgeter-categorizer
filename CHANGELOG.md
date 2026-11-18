# Changelog

Changes in this fork (Budgeter-categorizer) since the original Budgeter 2.0.0. Changes are in `app/index.html` unless an entry says otherwise. After updating it, run `bash install.sh` again, because the installed Mac app is a copy made when it's built.

## 2.4.0 — 2026-11-17

### Added
- Backups include `appVersion` and a `check` block: for each month, how many transactions it holds and their total in cents. The file is still `version: 2`, so older copies of Budgeter can read it.
- Restoring a backup first downloads `budgeter-before-restore-<date>.json`, a copy of what was there before. A browser or the desktop app can't confirm that a download was written (a save dialog may have been cancelled), so you're asked to check it saved before the restore goes ahead.
- **Undo** for 8 seconds after deleting a transaction with ×.

### Changed
- Restore checks every value in the file before anything changes: ids, dates, numbers, schedules and account types. One problem refuses the whole file and lists what's wrong. A 2.4.0 backup whose spending doesn't match its own `check` block is refused. Anything older versions could have written still restores, including long notes and the impossible transaction dates the old importer let through.
- Restore says so when only part of a backup could be saved (for example when the page's storage is full), instead of "Backup restored".
- Import reads amounts strictly. "12,50", "1.234,56" and "1e3" are skipped instead of being read as 1250, 1.23 and 13, and so are amounts of a billion or more. Dates that don't exist (February 30th), years outside 1900–2199, and month-name dates without a year are skipped.
- **System fonts.** The app no longer loads Google Fonts, so it makes no network requests at all; only the desktop app talks to Ollama on your own computer.
- **Electron 45.2.3** (was 33.2.1, which no longer gets security fixes) and electron-builder 26.18.1. The Mac app now needs macOS 13 (Ventura) or newer.
- New bills, accounts, goals and transactions get UUIDs (36 characters). Existing short ids keep working.
- Size limits: 50 MB for backups, 20 MB for CSV files, 50 MB for PDFs.
- 2027 HSA limits: $4,500 self-only, $9,000 family.
- Desktop app (`main.js`): only https links open in your browser. Other links are ignored.

### Fixed
- Saved data that can't be read (damaged or cut off) no longer loads as a blank plan that the next change saves over. A note at the top explains, saving stops, and Backup is off (it would hold a blank plan). You can save a copy of the damaged data, and **Set it aside**: nothing is deleted, it's kept on the device under its own name, and the app starts that part empty so you can restore a backup.
- Old Monthly planner (v1) data on the same computer is copied over only once, and never over newer data.
- Clearing a balance box no longer saves a balance of 0 and a net worth snapshot. The old value comes back.
- "Today" moves on after midnight when the app is left open.
- A merchant, category or tag named "constructor", "toString" or "__proto__" is counted and remembered correctly.
- Values from a backup file are escaped everywhere they're shown.
- A crafted backup with extreme years can no longer freeze the app at every start.
- A broken amount saved by an older version no longer stops every later import.

### Data
- New ids are UUIDs, and backups gain `appVersion` and `check`. Older versions ignore both and can restore a 2.4.0 backup, but they don't verify the `check` block.
- New localStorage keys: `budgeter:moved-v1` (the v1 copy has been dealt with) and `budgeter:unreadable:<where>:<key>:<time>` (data that was set aside).

### Known limits
- Started with Electron 45 on Windows, but not yet run against a real model or on the Mac. See `tests/manual-checklist-2.4.0.md`.
- Two deletes in quick succession stack their Undo messages: the newer one covers the older until it goes.

## 2.3.0 — 2026-11-12

### Added
- Approve AI picks one at a time or in bulk. On the Spending tab, each transaction with an AI-picked category has a checkbox at the right end of its row. They all start ticked: untick the wrong ones and click **Approve N AI picks**.
- An **AI** select-all checkbox in the table header, at the right end. It shows a dash when only some picks are ticked.

### Changed
- **Approve** replaces **Accept all**, and it only approves the ticked picks. Approved picks lose their AI badge and go into merchant memory. Unticked picks keep their badge and stay unticked, so they're easy to find and fix.
- Select-all and Approve only cover the rows you can see. A search, or the "AI picks to review" filter, narrows what gets approved.
- The Approve button is a regular button instead of a ghost button, since it's the main action while reviewing.

## 2.2.0 — 2026-11-12

### Changed
- AI category sorting should be about 2.5× faster. This is estimated from the shorter answers and not yet measured on a real model. The model now answers `{"1":"Groceries","2":"Shopping",…}` instead of `{"picks":[{"n":1,"category":"Groceries"},…]}`. The output used to take about 600 tokens per batch of 30 merchants (about 31 s with qwen3-vl:8b on an M1 Pro), and writing the answer was nearly all of that time. The numbered keys still tie each answer to its merchant. Answers in the old format are still read.
- Sorting results are applied batch by batch instead of all at once at the end:
  - **Import window:** picks appear in the preview as each batch finishes. Clicking Import part way keeps the finished picks.
  - **Spending tab:** each batch is saved as soon as it's back, so stopping, quitting the app or an error keeps what's done.

### Added
- A **Stop sorting** link in the import window.
- On the Spending tab, the Sort button becomes **Stop sorting (30 of 51 done)** while it runs. A stop takes effect after the batch in progress. Pressing **Sort N uncategorized** again carries on with the transactions that are still uncategorized.
- A message when the AI can't place any of the merchants it was sent. Before, nothing was shown at all.

### Fixed
- Closing the import window or clicking Import no longer leaves the AI sorting the remaining batches in the background.

## 2.1.0 — 2026-11-12

### Added
- **AI category sorting.** Your local model sorts the money-out transactions that your merchant memory and rules don't cover. Order of sorting:
  1. Merchants you've sorted before (exact text).
  2. Your keyword rules.
  3. Merchants that match one you've sorted before once cleaned up (see "Merchant memory" below).
  4. The AI.

  The AI:
  - can only answer with your own categories, Transfer or Uncategorized (enforced by the schema on Ollama, and checked again in the app);
  - is shown up to 40 of your past choices and rules as examples;
  - is told to answer Uncategorized rather than guess;
  - gets descriptions in batches of 30.

  It runs automatically on import, and from a **Sort N uncategorized** button on the Spending tab for the month you're viewing.
- **Review AI picks.** AI-picked categories get an "AI" badge, there's an "AI picks to review" filter, and an **Accept all** button. Accepted picks go into merchant memory, so the AI is needed less over time. Changing a category by hand clears its badge.
- **Model picker.** A Model menu in the import window lists the models installed in Ollama. Your choice is saved, and the app falls back to gemma3:4b if the chosen model is removed. Screenshots need a model that reads images, such as gemma3, gemma4 or qwen3-vl.

### Changed
- **Merchant memory** also matches on a cleaned-up name: letters only, filler words such as "SQ", "POS" and "LLC" dropped, first two words kept. So "SQ *BLUE BOTTLE 0977 BERKELEY CA" matches an earlier "SQ *BLUE BOTTLE 0412 OAKLAND CA".
- Setting a category also applies it to the other charges that month from the same merchant that are still uncategorized or AI picks.
- The import preview shows every row instead of the first six. Each row has a checkbox to skip it and a category menu. Categories you change there go into merchant memory when you import.
- "Uncategorized" is no longer saved to merchant memory.

### Fixed
- Two identical charges on the same day (for example two $5 coffees) both import. Before, the second was skipped as a duplicate. Re-importing an overlapping statement still skips what's already logged, and rows repeated across overlapping screenshot pieces are still merged.

### Data
- Transactions can have a new optional `ai: true` field, and the chosen model is saved under the `budgeter:model` key. No migration is needed: older versions ignore both, and backups restore in either direction.

### Known limits
- Tested in a browser with a simulated Ollama, not with a real model. How accurately it sorts depends on the model.
- **Sort N uncategorized** works on one month at a time.
- Ollama keeps recent prompts in memory as a cache (about 4 GB was seen after 20 sorting batches, with a limit of 8 GB). It's freed when the model unloads, or right away with `ollama stop <model>`.
