# Budgeter 2.4.0: manual test checklist

Run this before calling 2.4.0 done, and again on each machine (Windows, Mac). Each step says what to do and what you should see. Tick the box only when you saw it yourself. In Phase 3 these become automated tests.

**Before you start**
- Use a browser profile or app with no real data in it, or download a backup first.
- The fixture backups are in `tests/fixtures/backups/`:
  - `good-2.3.json`: a normal backup, as Budgeter 2.3 writes it.
  - `malformed.json`: wrong types, missing fields, a month numbered 13, `done: null`.
  - `malicious.json`: values written to run script if they were ever put into the page unescaped, plus `__proto__` keys.
- Browser preview: `python3 -m http.server 8765 --directory app` from the repo root (on Windows `python` may be the command), then open http://127.0.0.1:8765. Desktop app: `npm start`.

## 1. Backups

- [ ] **A 2.3 backup restores.** Restore from backup → `good-2.3.json` → Restore. You see "Backup restored", and the Accounts tab lists Checking, High-yield savings, Visa, Car loan, HSA and Brokerage.
- [ ] **A safety copy is saved first.** Restoring also downloads `budgeter-before-restore-<today>.json` before anything changes.
- [ ] **It exports again identically.** Download backup. Open the new file next to `good-2.3.json`: `plan` and `tx` hold the same values (key order and the `_ts` number may differ).
- [ ] **The export has the new fields.** The file has `"version": 2`, `"appVersion": "2.4.0"` and a `check` block: `2026-10` has count 5 and cents -26653; `2026-11` has count 6 and cents -6023.
- [ ] **A 2.4.0 backup restores.** Restore the file you just downloaded. You see "Backup restored".
- [ ] **A tampered backup is refused.** In a copy of that file, change one transaction's `amt` (for example -64.18 to -64.19) and restore it. You see "This backup was not restored", and it says spending for that month doesn't match the file's own check. Nothing changes.
- [ ] **A malformed backup is refused.** Restore `malformed.json`. You see "This backup was not restored" with a list of problems. Everything on every tab looks as it did before.
- [ ] **A malicious backup is refused.** Restore `malicious.json`. Same as above. No alert box appears, now or after reloading.
- [ ] **A huge file is refused.** Restore any file over 50 MB. You see a message, and nothing changes.
- [ ] **An old Monthly planner (v1) backup still restores**, if you have one.

## 2. Unreadable saved data

To simulate damage, open the browser's developer tools (F12) → Console, and run the line shown. Then reload.

- [ ] **A cut-off plan.** `localStorage.setItem('budgeter:plan', localStorage.getItem('budgeter:plan').slice(0, 500))`. After the reload, a red note at the top says some saved data can't be read. It does not show a blank "Nothing here yet" page with no explanation.
- [ ] **Nothing is saved over it.** Change something (a category budget, say). The status at the top right says "Not saved". In the console, `localStorage.getItem('budgeter:plan').length` is still 500.
- [ ] **Restore waits.** Restore from backup shows a message asking you to deal with the unreadable data first.
- [ ] **Save a copy** downloads `budgeter-unreadable-<today>.json`, holding the damaged text exactly as stored.
- [ ] **Remove it** asks first, saves the copy, removes the damaged data and reloads. The note is gone, and you can restore a backup.
- [ ] **A cut-off month.** Same as above with `budgeter:tx-<this month>` (for example `budgeter:tx-2026-11`). The note names that month.

## 3. Import (finding 5)

Import statement → paste each line below under a header line `Date,Description,Amount` → Read as CSV.

- [ ] `11/02/2026,Coffee,12.50`, `11/02/2026,Coffee,"1,234.56"`, `11/02/2026,Coffee,(5.00)` and `11/02/2026,Coffee,-5` are each read: you see $12.50, $1,234.56, −$5.00 and −$5.00 (with the sign setting on "Negative amounts are money out").
- [ ] `11/02/2026,Coffee,"12,50"` and `11/02/2026,Coffee,1e3` are **not** imported as 1250 or 1000. The row is skipped.
- [ ] `02/30/2026,Coffee,5.00` is skipped: there is no February 30th.
- [ ] A CSV file over 20 MB and a PDF over 50 MB are refused with a message.

## 4. Everyday use

- [ ] **Undo a delete.** On Spending, press × on a transaction. It disappears, and a message with an Undo button appears. Undo brings it back in the same place. Without Undo, it stays deleted after a reload.
- [ ] **A blanked balance is ignored.** On Accounts, clear a balance box and press Tab. The old balance comes back, and the "since" date under it doesn't change. Typing 0 sets it to 0.
- [ ] **New ids are UUIDs.** Add a bill, then Download backup: its `id` looks like `3f2c9a1e-…` (36 characters). Bills, accounts and transactions you had before keep their short ids and still work.
- [ ] **A merchant called "constructor".** Log $5 at merchant `constructor` with category Groceries. It shows under Groceries, and the Spending total adds the $5.
- [ ] **Overnight.** Leave the app open past midnight (or change the computer's date). Within a minute the calendar's highlighted day and "Today" move to the new date.
- [ ] **HSA limit.** With an HSA account, Accounts shows the 2027 limit ($4,500 self-only, $9,000 family) once the computer's date is in 2027.

## 5. Links (finding 9, desktop app only)

- [ ] On Spending → Import statement with Ollama turned off, the Ollama link opens https://ollama.com in your default browser, not inside Budgeter.
- [ ] Links that aren't https never open. You can't easily click one in the app; this is checked by reading `main.js`.

## 6. Looks and layout

- [ ] Every tab opens with no errors in the console (F12). The only allowed error is a missing `favicon.ico` in the browser preview.
- [ ] Light and dark themes both look right (switch your system theme).
- [ ] At phone width (640 px or less; in developer tools, toggle the device toolbar), every tab fits with no sideways scrolling.
- [ ] In the browser preview (no `window.claude`), everything except the AI features works and saves ("Saved on this device").

## 7. Your tests (need your machines)

- [ ] **Real-model sorting.** Desktop app with Ollama running and a model pulled (the default is `gemma3:4b`; run `ollama list` to see what you have). Import a real statement CSV with uncategorized merchants and press Sort. Write down how many merchants it sorted, how long it took, and how many picks were wrong.
- [ ] **`npm start` on the Windows PC.** `npm install` then `npm start`. The window opens, every tab works, and Download backup asks where to save.
- [ ] **The Mac.** `bash install.sh`, then repeat sections 1, 4 and 6 in the installed app.
