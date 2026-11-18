# Budgeter desktop: review and migration plan

Written 2026-11-16 from a planning session. Nothing in this plan has been built yet.

The goal is to turn Budgeter (one HTML file, wrapped in Electron for the Mac) into a lightweight desktop app for macOS, Windows and Linux, with encrypted local storage and sync between the owner's own devices. The storage, change-log, sync and audit patterns are meant to be reused later in a teacher app that handles IEP data (FERPA/IDEA).

## Contents

1. [Decisions already made](#1-decisions-already-made)
2. [Answers that shaped the plan](#2-answers-that-shaped-the-plan)
3. [Who does what](#3-who-does-what)
4. [Code review of the current app](#4-code-review-of-the-current-app)
5. [Shape of the new app](#5-shape-of-the-new-app)
6. [Phases](#6-phases)
7. [Reuse for the IEP app](#7-reuse-for-the-iep-app)
8. [Risks and open questions](#8-risks-and-open-questions)
9. [First step](#9-first-step)

## 1. Decisions already made

These are fixed unless a serious problem turns up.

- **Framework:** Tauri 2 (web front end, Rust backend). Chosen over Electron and Wails.
- **Front end:** keep the existing HTML and JavaScript as much as possible.
- **Storage:** SQLite in WAL mode, saving every change immediately.
- **Money:** integer cents, never floating point; formatted only for display.
- **Encryption at rest:** SQLCipher or comparable, with the key in the OS keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service). No hard-coded keys.
- **Sync:** between the owner's own devices, with no third-party cloud. Each device runs a small local HTTP server; devices exchange changes over Tailscale (possibly Headscale later).
- **Sync design:** UUIDs, tombstones, an updated-at timestamp and device id on each record, an append-only change log, an audit trail that doubles as undo history. Start with last-writer-wins with preserved history; CRDTs are a later step.

## 2. Answers that shaped the plan

- **Baseline is 2.3.0.** `app/index.html` is currently at 2.1.0. The 2.2.0 and 2.3.0 patches are committed as `.patch` files in the repo root but were never applied. `budgeter-2.3.0.patch` is cumulative: it gives the same `app/index.html` as the 2.2.0 patch does, and fails if 2.2.0 is applied first. Neither version has been tested against a real model. `CHANGELOG.md` describes 2.3.0 and `package.json` says 2.0.0.
- **No existing data needs migrating.** Statements can be re-imported.
- **A 2.4.0 stopgap comes first** (Phase 0): the single-file app with the cheap review fixes and an export the new app can read. It is the working app while the new one is built, so the new app needs a tested importer for its backups.
- **The new app is a fork.** The old single-file app (browser and Mac app) keeps working as it is and is not compatible with the new project.
- **Devices:** Windows PC, Mac, Linux, family machines. iPhone access is a wish-list item, by viewing over Tailscale rather than an App Store app.
- **Threat model:** a stolen or lost device. Backups are a file encrypted with a passphrase the owner chooses.
- **Local AI sorting is a core feature** and returns right after the port, before storage work.
- **Restructuring:** gradual split into plain ES modules. No TypeScript, no framework.
- **Conflict rules:** written as SQL plus tests by the owner, with Rust around them written by AI.
- **Distribution:** open source on GitHub, own and family machines first. No paid signing or notarization.

## 3. Who does what

Every phase splits its work three ways.

- **AI writes, owner reviews:** boilerplate and code that is hard for a beginner but should still be readable: the Rust shell, Tauri config, keychain and encryption plumbing, the sync server, CI and packaging. Each phase lists questions the owner should be able to answer before accepting the code.
- **Owner writes, with guidance:** the database schema, money handling, the sync protocol and conflict rules, front-end changes and tests. Guidance means hints and checkpoints, not finished code.
- **Decisions:** listed with a recommendation; the owner chooses.

The owner's strongest language is JavaScript. Rust is new, SQL is being learned, and unit testing is new.

## 4. Code review of the current app

Reviewed: `app/index.html` at 2.1.0, `main.js`, `preload.js`, `package.json`. Line numbers refer to that version and will shift once the patches are applied. Findings are ranked most severe first, and Phase 0 refers to them by number.

### Findings

**1. High: a corrupt save is treated as "no data" and then overwritten.**
- **Where:** `lget` at `app/index.html:257`, used by the startup load at 1610 and `ensureTx` at 339.
- **Risk:** a JSON parse error returns `null`, the app starts with a blank plan or an empty month, and the next edit saves over data that might have been recoverable. Desktop data exists only in localStorage, and backups are manual.
- **Fix:** tell "missing" apart from "unreadable" and refuse to save when unreadable. The existing `S.locked` guard does this, but only for account errors. SQLite plus automatic rotating backups replaces the rest.

**2. High: saves replace whole documents, and multi-step changes aren't atomic.**
- **Where:** `persist` at 276, `sRead` at 263, `restore` at 332, `editTx` at 719.
- **Risk:** device and account copies are compared by one wall-clock `_ts` per document, so the later writer replaces the other's entire plan or month. Restore overwrites the plan and the months in the file but leaves other months alone, with no snapshot taken first. Moving a transaction to another month is a delete-and-save followed by an add-and-save.
- **Fix:** database transactions, a snapshot before restore, and per-record changes.

**3. High for the project's goals: the code is hard to read and impossible to unit test.**
- **Where:** all the JavaScript sits in one closure, lines 200 to 1619.
- **Risk:** single-letter names (`pn`, `pd`, `occ`, `mi`, `r2`), lines hundreds of characters long, and no tests. Nothing can be imported into a test.
- **Fix:** move the pure logic (schedules, parsing, payoff maths) into modules and put tests around it before touching storage.

**4. Medium: money is floating-point dollars.**
- **Where:** `r2` at 208, `sum` at 212, `fundGoal` at 752, the "actual amount" handler at 1584 (unrounded, while `setDue` at 515 rounds).
- **Risk:** `r2(1.005)` returns `1`, and `0.1 + 0.2` gives `0.30000000000000004`. Today the errors are under a cent and hidden by display rounding, with tolerance checks such as `> bud + 0.5` papering over them.
- **Fix:** integer cents. Parse text straight to cents rather than multiplying a float by 100, and write down the rounding rule for interest.

**5. Medium: import parsing is silently wrong outside US formats.**
- **Where:** `pn` at 788, `pd` at 789, the row filter at 988.
- **Risk:** `"12,50"` parses as 1250 and `"1.234,56"` as 1.23. Day-first dates are swapped when the day is 12 or under and dropped otherwise. Rows that fail to parse vanish without a count.
- **Fix:** a strict amount grammar, a day range check, and an "N rows couldn't be read" line in the preview.

**6. Medium: a crafted backup can run script, and a malformed one can brick the app.**
- **Where:** `restore` at 328 only checks that three arrays exist. Unescaped values include ids in `data-id` (1315, 1241, 1251, 1465), pay dates at 1147 and 1269, recorded amounts at 1241, and the model name at 859 and 1432.
- **Risk:** these values are normally app-generated, but a backup file can set them to anything, and there is no Content-Security-Policy. A backup with `done: null` is saved before rendering, so the app fails on every start.
- **Fix:** validate everything at the boundary (id shape, ISO dates, finite numbers), escape every interpolation regardless, and add a CSP. Sync payloads from another device need the same boundary.
- **Not found:** any XSS path from typing, CSV import or AI output. Those are escaped consistently, and there is no `eval`.

**7. Medium: ids are weak and some records have none.**
- **Where:** `uid` at 203; occurrence keys at 396 and 496.
- **Risk:** ids are seven characters from `Math.random`, never checked for collisions, and delete and edit act on every match. Categories are keyed by name, and per-occurrence state by strings such as `out:<id>:<date>`.
- **Fix:** `crypto.randomUUID()` and real keys in the schema.

**8. Medium: destructive actions have no undo.**
- **Where:** transaction delete at 1522, the balance handler at 1592, `del` at 695.
- **Risk:** the × on a transaction deletes with no confirmation. Clearing a balance field stores 0 and records a net-worth snapshot. Deleting an account leaves bills, transactions and goals pointing at nothing.
- **Fix:** tombstones and undo, and foreign keys.

**9. Medium: Electron shell.** Most of this disappears with Tauri, but the same decisions come back there.
- Electron is pinned at 33.2.1 (`package.json:18`), released in late 2024 and no longer supported, so its Chromium is unpatched.
- `shell.openExternal` accepts any URL scheme (`main.js:19`).
- PDFs are parsed in the privileged main process (`main.js:99`).
- Any `file://` page passes the sender check (`main.js:30`).
- `OLLAMA_HOST` can point off-machine over plain HTTP while the UI says nothing leaves the computer (`main.js:29`).

**10. Low to medium: Google Fonts loads on every launch.**
- **Where:** `app/index.html:7`.
- **Risk:** each start tells Google the user's IP address, and the external stylesheet blocks a strict CSP.
- **Fix:** bundle the two fonts or use system fonts.

**11. Low: smaller items.**
- "Today" is computed once at line 213, so an app left open overnight uses yesterday's date.
- A merchant named `constructor` gets an undefined category, because plain objects are used as maps (542, 707).
- The byte-order-mark regex at 797 holds an invisible character an editor could strip.
- HSA limits stop at 2026 (243).
- Import and restore have no file size limit.
- The README says `cd Budgeter` after cloning `Budgeter-categorizer`.

### What makes the port easy or hard

**Easy:**
- No framework and no build step, so Tauri can serve `app/` as it is.
- Storage already goes through a handful of functions (lines 257 to 284).
- Clicks are routed by `data-act`, which maps well onto named commands.
- The Electron bridge is five methods.

**Hard:**
- About 39 call sites change `plan` or `tx` in place and then call `savePlan()` or `saveTx()`, so the storage layer never learns what changed. A change log and sync both need that.
- `plan` is one document mixing records, settings, per-occurrence state, history and merchant memory, which all has to become tables.
- `<a download>` (line 294) is unreliable in Tauri's webviews, and Ollama and PDF reading need a new home. The port therefore needs a small platform adapter.

### Worth keeping

- Consistent escaping, with AI output treated as untrusted.
- Schema-constrained AI answers, with the app setting signs and doing all arithmetic.
- The versioned backup format and `upgrade()`.
- Dates stored as local ISO strings, with day arithmetic done in UTC.
- Count-based duplicate detection and the import preview.
- Accessibility: labels, keyboard reordering, light and dark tokens.
- The schedule, holiday, projection and payoff logic, which should move over intact with tests around it.

## 5. Shape of the new app

- **Front end:** the existing HTML and JavaScript, split into plain ES modules, calling one `api` module. Keeping that module as the only route to the backend is what later lets a phone browser use the same front end over the tailnet.
- **Backend:** Rust owns the database (rusqlite with SQLCipher), the keychain, the Ollama calls and the sync server. Encryption plus a sync server that writes to the database both push database access into Rust.
- **Writes:** the page sends a list of row changes, never SQL. Rust checks each change against a list of allowed tables and columns.
- **How the existing code survives:** the page keeps its in-memory `plan` and `tx`. On each save, a `diff(previous, current)` function turns the difference into row changes, sent to Rust as one transaction. The 39 save call sites stay as they are.

### How the phases differ from the original six

- **Phase 0 is new:** the 2.4.0 stopgap.
- **Phase 2 is new:** local AI returns early.
- **Cents come before SQLite** (Phase 3), so the first tests are in JavaScript on small pure functions.
- **A bare CI build moves up to Phase 1**, so every later phase is known to build on all three systems.
- **Encryption stays after SQLite**, with one rule: the new app holds test data only until Phase 5 is done. That keeps the database inspectable while the schema is being learned and avoids converting a plaintext database later.

## 6. Phases

Each phase ends with a working app.

### Phase 0: Budgeter 2.4.0 stopgap (single file)

- **Goal:** today's app at 2.3.0, plus the cheap review fixes, plus an export the new app can trust.
- **In:**
  - Apply `budgeter-2.3.0.patch` alone: `git apply --exclude=CHANGELOG.md budgeter-2.3.0.patch`. It also sets `package.json` and `package-lock.json` to 2.3.0. Its `CHANGELOG.md` part is skipped because that file already exists.
  - Refuse to save over unreadable data (finding 1).
  - Validate restored backups and escape the remaining values (6).
  - Strict amount parsing and a "N rows couldn't be read" count (5).
  - Undo on transaction delete, and ignore a blanked balance (8).
  - `crypto.randomUUID()` for new ids (7).
  - Automatic backup download before a restore (2).
  - The small items (11), and https-only external links (9).
- **Out:** cents, modules and a CSP. Those belong to the fork.
- **Export:** stays backup `version: 2` so older copies can read it, and gains `appVersion` and a `check` block (per month: item count and total in cents).
- **Done when:**
  - A 2.3 backup restores and re-exports identically.
  - A corrupted saved value shows an error instead of a blank plan.
  - A malicious backup fixture is rejected.
  - AI sorting has been run against a real model.
  - It runs with `npm start` on the Windows PC.
  - Light and dark themes, the phone layout and the no-`window.claude` fallback still work.
- **Concepts:** trust boundaries, input validation, reading a diff.
- **Tests:** a written manual checklist and three fixture backups (good, malformed, malicious). They become automated tests in Phase 3.
- **AI writes, owner reviews:** all of it. Questions to answer before accepting:
  - What happens now if saved data is half a JSON string?
  - Through which places can outside data enter the app?
  - Why does the backup stay at version 2?
- **Owner writes (optional):** the skipped-rows count. Hint: compare the row count before and after the filter in `recompute`.
- **Decisions (made 2026-11-17):**
  - Electron upgraded in 2.4.0, to 45.2.3. Tested on Windows; the Mac still needs a test, and needs macOS 13 or newer.
  - System fonts, so nothing is sent to Google. Phase 1 has no fonts to bundle.
  - claude.ai is not a host for this fork, so there is nothing to republish. The `window.claude` code paths are inherited from the original project and stay dormant.

### Phase 1: Tauri shell

- **Goal:** the 2.4.0 front end in a Tauri 2 window on Windows, macOS and Linux. Storage is unchanged, with test data only.
- **Work:**
  - Move the inline script and styles into files so a strict CSP is possible.
  - Fonts: nothing to bundle; 2.4.0 already uses system fonts.
  - Add a `platform.js` adapter for saving files through a native dialog.
  - Set up GitHub Actions to build on all three systems.
  - Run a spike proving SQLCipher and the keychain library compile on all three.
- **Done when:**
  - CI is green with downloadable builds.
  - Every tab works.
  - Backup and calendar files save through the dialog.
  - The app makes zero network requests.
  - An injected inline script is blocked.
- **Concepts:** webview versus Rust core, commands, capabilities, CSP, why system webviews differ.
- **Tests:** a smoke checklist per system, and one Rust test to prove `cargo test` runs in CI.
- **Security:** minimal capabilities, no remote content, external links limited to https.
- **AI writes:** the Rust shell, config, capabilities, CI and adapter. Questions:
  - If a script were injected into the page, what could it call?
  - What changed to satisfy the CSP?
  - What does each capability grant?
- **Owner writes:** the move of script and styles into files (the app must look identical afterwards), and the smoke checklist.
- **Decisions:**
  - New repository or a branch? Recommended: a new repository.
  - The app identifier. Choose it once: the data folder depends on it.

### Phase 2: Local AI

- **Goal:** sorting, screenshots, pasted text, PDFs and Ask all work, behind the same five-method `window.budgeterDesktop` interface. The prompts, batching, review screens and checks in the page carry over untouched; only the bridge in `main.js` is replaced.
- **Work:** Rust commands for status, JSON answers and streamed chat, and pdf.js running inside the page.
- **Done when:**
  - 30 or more merchants sort with a real model on Windows and Mac.
  - Stop works.
  - With Ollama off, the setup message shows.
  - The page itself still cannot reach the network; only Rust can, and only to this machine.
- **Concepts:** async Rust, HTTP clients, streaming.
- **Tests:** Rust tests against a fake Ollama covering success, missing model, timeout and invalid JSON.
- **Security:** the Ollama address is fixed to this machine unless a setting says otherwise, and the UI states where data goes. Model output stays untrusted.
- **AI writes:** the commands and bridge. Questions:
  - What does the page receive when Ollama is off?
  - What happens to a running request on Stop?
  - Where is the model name checked?
- **Owner writes:** a real-model accuracy log on 50 of their own merchants. This is the first measurement of the 2.2 speed claim.
- **Decisions:** pdf.js in the page (recommended, same library as today) or a Rust PDF library.

### Phase 3: Modules, first tests, money in cents

- **Goal:** the pure logic lives in modules with tests, and every stored and computed amount is an integer number of cents.
- **Work:** modules for money, dates and schedules, parsing, categorizing, projection, debt, storage and views. A test runner. The model converted to cents. A backup importer that converts version 2 (dollars) to cents and verifies the `check` block.
- **Done when:**
  - `npm test` passes in CI.
  - No `parseFloat` or `toFixed` exists outside the money and parsing modules.
  - A fixture plan gives the same Overview, Accounts and Debt figures before and after conversion.
  - Importing a 2.4.0 backup reproduces its `check` totals exactly.
- **Concepts:**
  - What a unit test is: arrange, act, assert; one behaviour per test; the name states the rule.
  - "Golden" tests that pin current behaviour.
  - Pure functions.
  - Rounding rules.
  - The .NET testing book maps across directly: a `[Fact]` is a `test()`, and `Assert.Equal` is `assert.equal`.
- **Tests the owner writes:**
  - Amounts that should parse: `"12.50"`, `"12.5"`, `"1,234.56"`, `"(5.00)"`, `"-5"`, `".5"`.
  - Amounts that should be rejected: `"12,50"`, `"1e3"`, an empty string.
  - Formatting: negatives, zero, thousands separators.
  - Schedules: monthly on the 31st in February, every-2-weeks across a clock change, leap years, end dates.
  - CSV with quotes, a byte-order mark and Windows line endings.
  - The duplicate-count rule on import.
- **AI writes:** the test runner setup, the move-only module split, and the golden-test harness. Questions:
  - How do you run one test file?
  - What would you do when the golden test fails?
- **Owner writes:** `money.js`, test first, then the conversion one area per commit (accounts, bills and income, transactions, goals, projections), renaming terse identifiers along the way.
- **Hints:**
  - List the tests in plain English first.
  - Watch each test fail before making it pass.
  - Parse the text into sign, whole and fraction; never multiply a float by 100.
  - Projections may compute in floats, but round through one function at the points where money is posted.
- **Decisions:**
  - Node's built-in test runner (recommended, nothing to install) or Vitest.
  - The rounding rule for interest.

### Phase 4: SQLite

- **Goal:** real tables in WAL mode, with each save one transaction. localStorage keeps only interface preferences.
- **Work:** the schema, numbered migration files tracked by `PRAGMA user_version`, Rust commands to load everything and to apply a batch of changes, `diff()` in the front end, the backup import, and automatic rotating backups.
- **Done when:**
  - Killing the app mid-use loses nothing completed, and `PRAGMA integrity_check` passes.
  - Moving a transaction between months is one `UPDATE`.
  - A 2.4.0 backup imports and a SQL `SUM` matches its `check` totals.
  - A database from a newer app version is refused.
- **Concepts:** tables, keys and foreign keys, transactions, WAL, migrations, indexes, parameterized queries.
- **Tests:**
  - `diff` cases: add, change, delete, reorder, no change.
  - Migrations from every older fixture.
  - Constraints rejecting bad rows.
  - Import round trip.
- **Security:** only parameterized SQL. Rust checks every change against a list of allowed tables and columns. No real data goes in yet.
- **AI writes:** the Rust database module, the migration runner and rotating backups. Questions:
  - What happens if change 3 of 5 fails?
  - What are the `-wal` and `-shm` files, and what do they mean for copying the database?
  - Where is the list of tables and columns the page may write?
- **Owner writes:** the schema, `diff()` with tests, and the backup-to-rows import.
- **Checkpoints:**
  1. A paper sketch of the tables is reviewed before any SQL. Starting list: accounts, categories, transactions, bills, income, goals, goal contributions, occurrences (replacing the `done`, `actual` and `payOn` maps, keyed by item and date), merchant memory, rules (one row each), settings, net-worth snapshots.
  2. Every table has a UUID key and the `updated_at`, `updated_by` and `deleted_at` columns reserved for Phase 6.
  3. Five queries the app needs are written to test the design.
- **Decisions:**
  - Store goal savings as contribution rows rather than a running total. Recommended: rows, because they merge cleanly in sync.
  - `synchronous = FULL` or `NORMAL`. Recommended: `FULL`; write volume is tiny and durability is a stated goal.

### Phase 5: Encryption

- **Goal:** a SQLCipher database with a random key in the OS keychain, plus a passphrase-protected backup. Real data moves in after this phase, not before.
- **Done when:**
  - A known merchant name can't be found in the database files.
  - With the keychain entry deleted, the app offers a restore and never creates an empty database over the old one.
  - A backup made on Windows restores on the Mac.
  - A wrong passphrase fails cleanly.
- **Concepts:** at rest versus in transit, threat modelling, random keys versus passphrase-derived keys, authenticated encryption, key loss.
- **Tests:** wrong key fails, create-close-reopen, backup round trip, a tampered backup is rejected, and a manual keychain matrix per system.
- **Security:** the key is never logged and never reaches the page. Temporary data stays in memory. Rotating backups share the database key, so they are useless if the keychain is lost; the passphrase backup is the recovery path.
- **AI writes:** the keychain and open sequence, backup encryption, and a Linux fallback when no keyring service runs. Questions:
  - Your laptop is stolen asleep but logged in; what protects the data?
  - What do you need to read a backup file?
  - What happens on a new PC with no keychain entry?
- **Owner writes:** a one-page threat model (reusable for the IEP app) and the test matrix.
- **Decisions:**
  - Keep a plaintext JSON export as an escape hatch? Recommended: yes, clearly labelled.
  - The backup file format. Recommended: a standard one (age) so the file can be opened without this app.

### Phase 6: Change log, history, undo

- **Goal:** every write appends to an append-only `changes` table in the same transaction. Deletes become tombstones. Each record has a history view, and changes can be undone.
- **Starting sketch for a change:** its own id, a per-device sequence number, the device id, a logical-clock timestamp, table, row id, operation, old and new values, the actor, a batch id grouping one user action (the undo unit), the schema version, and a pointer to the change it replaced.
- **Done when:**
  - Replaying the log from empty rebuilds the tables exactly.
  - Undo restores the previous state and is itself logged.
  - The log rejects edits and deletes.
  - A deleted transaction disappears from views but remains as a tombstone.
- **Concepts:** event log versus current state, idempotency, logical clocks, audit versus undo.
- **Tests:** the rebuild test, undo sequences, delete then undo, and a clock that still moves forward when the wall clock goes backwards.
- **Security:** the log holds old and new values, so it gets the same encryption. An `actor` column records the person, separate from the device.
- **AI writes:** the single write path, the hybrid logical clock, and test helpers. Questions:
  - Why can't `Date.now()` alone order changes from two devices?
  - What does undo write?
  - How would you prove the log is complete?
- **Owner writes:** the `changes` schema, apply-and-inverse SQL, the history view, and scenario tests.
- **Decisions:**
  - Record whole rows or only changed fields. Recommended: changed fields with old and new values.
  - Keep the log forever (recommended) or compact it.

### Phase 7: Sync over Tailscale

- **Goal:** devices converge, and conflicts are kept and shown.
- **Protocol:** paired devices greet each other with app version, schema version and device id. Each then pulls "changes since what I've seen from each device", applies them in pages through the same write path as local changes, and moves its cursor in the same transaction.
- **Done when:**
  - Three simulated devices with random edits and random sync order end identical.
  - A sync killed midway recovers without duplicates.
  - A clock a day ahead can't win forever.
  - Mismatched schema versions refuse politely.
  - An unpaired tailnet device is rejected.
  - Two edits to the same field keep the later one, with the other in history and on a "Conflicts to review" list.
- **Concepts:** version vectors, at-least-once delivery, the limits of last-writer-wins, authentication versus network membership.
- **Tests the owner writes:**
  - Edit and edit of the same field.
  - Edit and edit of different fields.
  - Edit and delete.
  - The same category name created on two devices.
  - An account deleted while another device adds to it.
  - A relay through a third device.
  - The same page applied twice.
  - A month offline, then sync.
- **Security:**
  - A token on every request, exchanged at pairing and kept in the keychain.
  - Listen only on the Tailscale address.
  - Size limits.
  - Remote changes pass the same checks as local ones.
  - A way to revoke a device.
- **AI writes:** the server, client, pairing and the multi-device test harness. Questions:
  - Sync dies after page 2 of 5; what state is each device in?
  - Why isn't being on the tailnet enough?
  - Two offline devices edit the same transaction; what does each end up with?
- **Owner writes:** the protocol document before any code, the conflict rules table, the merge SQL, and the conflicts screen.
- **Decisions:**
  - A manual Sync button first (recommended).
  - The delete-versus-edit rule. Recommended: delete wins, with the edit kept in history and flagged.
- **Where CRDTs fit:**
  - Per-field last-writer-wins is already the simplest CRDT, and it will have been built by hand here.
  - cr-sqlite automates that merge and is worth trying once it is clear what it replaces.
  - Automerge suits long text edited by two people, such as IEP notes.
  - Running totals are the weak spot of last-writer-wins, which is why goal savings become rows in Phase 4.

### Phase 8: Releases

- **Goal:** a version tag builds unsigned installers for all three systems, with checksums and install instructions.
- **Done when:**
  - Fresh installs are tested on each system, including the warning screens.
  - An upgrade keeps data and runs migrations.
  - A downgrade is refused safely.
- **Concepts:** build matrices, supply-chain hygiene, what signing proves, version numbers tied to schema versions.
- **Tests:** CI runs the JavaScript and Rust tests before building, and a migration test from every released schema version.
- **Security:** published checksums, minimal workflow permissions, pinned actions, a dependency update routine.
- **What unsigned means for users:**
  - **macOS:** approve the app under System Settings → Privacy & Security, or build from source.
  - **Windows:** SmartScreen shows a warning with a "Run anyway" option.
  - **Linux:** unaffected.
- **AI writes:** the workflows and draft docs. Questions:
  - What does a checksum prove, and what doesn't it?
  - What triggers a release build?
  - What stops a newer database being opened by an older app?
- **Owner writes:** the release checklist and final install docs.
- **Decisions:** no auto-updater at first (recommended).

### Later

- Phone view over the tailnet: one home machine serves the same front end to the phone's browser.
- An always-on sync node.
- Per-field merge, then a CRDT trial.

## 7. Reuse for the IEP app

- **Reusable:** a change log and sync core that knows only table, row and fields, kept in its own Rust library from Phase 6.
- **Add now:** an `actor` column for the person, separate from the device.
- **Dead ends to avoid:**
  - A log that can never be purged conflicts with record-destruction duties, so design a purge path.
  - Silent last-writer-wins isn't acceptable for legal documents, so make conflict policy settable per table.
  - The log records writes only, and student records usually need read-access logging too.
  - Identity must not rest on tailnet membership once more than one person is involved.

## 8. Risks and open questions

**Risks**
- SQLCipher is the likeliest build problem on Windows; the Phase 1 spike exists to find out early.
- Linux's webview lags the others; date inputs and dialogs need an early check.
- Phase 3 touches most of the code and is where the project could stall. The golden tests and area-by-area commits are the guard.
- `diff()` relies on every record having a stable id, which categories lack today.
- Sync only happens when two devices are awake together.
- A lost keychain plus a forgotten passphrase means the data is gone, by design.
- Two apps will hold data for a while. The move from 2.4.0 to the new app is one-way, at the end of Phase 5.

**Open questions**
- Does the family share one budget or keep separate ones?
- Which Mac (Apple Silicon or Intel)?
- Which Linux distribution?
- The new repository's name.

## 9. First step

Start Phase 0: apply `budgeter-2.3.0.patch`, then run the result with `npm start` on the Windows PC against Ollama, so 2.3.0 gets its first real-model test before anything is fixed on top of it. Run `ollama list` first to see which models that machine has; the app's default is `gemma3:4b`.

Who does each task is in [AGENT-PLAN.md](AGENT-PLAN.md). The brief for the lead session is [OPUS-BRIEF.md](OPUS-BRIEF.md).
