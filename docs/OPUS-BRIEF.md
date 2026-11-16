# Brief for Claude Opus 5.5: take over Budgeter

Repo: the folder this chat is opened on, a checkout of `Budgeter-categorizer` on branch `main`. Read `.env` (this machine's settings), `CLAUDE.md`, `docs/PLAN.md` and `docs/AGENT-PLAN.md` first. This brief is saved as `docs/OPUS-BRIEF.md`.

## 1. Goal

Ship Budgeter 2.4.0 (Phase 0 in `docs/PLAN.md`), then do the Opus rows of Phases 1 to 8 in `docs/AGENT-PLAN.md`.

Done when (Phase 0):
- A 2.3 backup restores and re-exports identically.
- A corrupted saved value shows an error instead of a blank plan.
- A malicious backup fixture is rejected.
- AI sorting has been run against a real model.
- It runs with `npm start` on the Windows PC.
- Light and dark themes, the phone layout (640px and under) and the no-`window.claude` fallback still work.

Phases 1 to 8 each have a "Done when" list in `docs/PLAN.md` section 6.

## 2. Next step

Run `git apply --exclude=CHANGELOG.md budgeter-2.3.0.patch` in the repo. Then confirm `app/index.html` contains "Stop sorting" and "Approve", `package.json` says 2.3.0, and the `budgeter` preview (`.claude/launch.json`, port 8765) loads with no console errors.

## 3. Decided

- **Tauri 2**, web front end plus Rust backend: lightweight. Rejected Electron (too heavy) and Wails/Go.
- **Keep the existing HTML and JavaScript**, split gradually into plain ES modules: the owner must be able to read it. Rejected TypeScript, frameworks and bundlers.
- **SQLite in WAL mode, every change saved immediately; money as integer cents:** durability and exact arithmetic.
- **SQLCipher with a random key in the OS keychain, plus a backup file encrypted with a passphrase the owner chooses:** the threat is a stolen or lost device.
- **Sync between the owner's own devices over Tailscale, each device running a small local HTTP server:** no third-party cloud.
- **Last-writer-wins with preserved history first:** simple to learn. Rejected starting with CRDTs (Automerge, cr-sqlite); they come later.
- **2.4.0 stopgap first:** a working app while the new one is built. It leaves out cents, modules and a CSP; those belong to the fork.
- **2.4.0 export stays backup `version: 2`** and gains `appVersion` and a `check` block (per month: item count and total in cents): older copies can still read it and the new app can verify it.
- **The new app is a fork;** the old single-file app keeps working. No existing data needs migrating.
- **Phase order:** local AI in Phase 2 (sorting is a core feature), cents in Phase 3 before SQLite in Phase 4 (first tests in JavaScript), encryption in Phase 5 with test data only until it is done.
- **The page sends row changes, never SQL.** A `diff(previous, current)` function keeps the 39 save call sites unchanged.
- **Who writes what:** AI writes the Rust shell, config, encryption plumbing, sync server and CI for the owner to review. The owner writes the schema, money handling, `diff()`, sync protocol, conflict rules, front-end changes and tests, with hints only.
- **No paid signing or notarization:** open source, no budget.
- **iPhone:** view over Tailscale later. Rejected a Tauri iPhone app (needs Xcode; expires after 7 days on a free Apple ID).
- **Patches:** apply `budgeter-2.3.0.patch` alone. Rejected applying `budgeter-2.2.0.patch` first; the 2.3.0 patch is cumulative and fails after it.

## 4. Facts

- **App:** one file, `app/index.html`, currently 2.1.0. `main.js` and `preload.js` are the Electron wrapper (Electron 33.2.1, Mac build only; `npm start` on Windows is untested). `package.json` says 2.0.0.
- **Patch check, done in a throwaway copy:** `budgeter-2.2.0.patch` and `budgeter-2.3.0.patch` give the identical `app/index.html`. The 2.3.0 patch also sets `package.json` and `package-lock.json` to 2.3.0 and tries to create `CHANGELOG.md`, which already exists.
- **Rules for changes** are in `CLAUDE.md`: `claude.use(...)` must tolerate `null`; localStorage prefix `budgeter:`; schema changes in `upgrade(p)`; light and dark tokens and the phone layout must keep working; the app, not the model, sets signs and does arithmetic.
- **Code review:** 11 numbered findings in `docs/PLAN.md` section 4. Line numbers there refer to 2.1.0 and shift after the patch.
- **Local AI:** Ollama at `127.0.0.1:11434`, default model `gemma3:4b`. Which models are pulled differs by machine: run `ollama list`.
- **This machine:** the project is developed on Windows, macOS and Linux. This machine's settings are in `.env` at the repo root (not committed; keys in `.env.example`). Find tool versions by running the tools (`node -v`, `rustc --version`), not from notes.
- **Git:** `origin` is `https://github.com/kleinbuild/Budgeter-categorizer.git`. `.gitattributes` keeps line endings consistent; don't convert them.
- **Web copy:** `https://claude.ai/artifact/JNg9Q5Q8WCvU4tBUzzCRXM`, capabilities `db`, `downloads`, `sample`, `user`.
- **Owner:** newer developer. JavaScript is strongest; Rust is new; SQL and unit testing are being learned. Wants to understand and own everything that ships.
- **Agents:** eight role files in `.claude/agents/`: `mentor`, `rust-core`, `security-reviewer` (Opus); `tauri-builder`, `frontend-engineer`, `test-harness` (Sonnet); `repo-clerk`, `acceptance-checker` (Haiku). If you can start subagents, use them as mapped in `docs/AGENT-PLAN.md`; if not, do the steps yourself in the same order.

## 5. Done here

- Full code review of `app/index.html`, `main.js`, `preload.js` and `package.json`.
- Owner's answers to nine planning questions, recorded in `docs/PLAN.md` section 2.
- `docs/PLAN.md`, `docs/AGENT-PLAN.md` and the eight agent files written.
- Patch behaviour verified (see Facts).
- `.env.example`, `.gitattributes` and a "Machines" section in `CLAUDE.md` added, so the repo works the same on every development machine.
- All of the above is committed on `main`. No app code has changed.

## 6. Ask me first

- **Commit, push or open a pull request:** I decide what goes into history.
- **Republish to claude.ai:** it is public-facing.
- **Install or upgrade software,** including Electron and pulling `gemma3:4b` (about 3.4 GB): it changes my machine.
- **Delete files,** including the `.patch` files: not reversible.
- **Create a repository or choose the app identifier:** the data folder depends on it.
- **Write finished code for my parts** (schema, money, `diff()`, conflict rules, protocol, tests): I am learning by writing them.
- **Change the fonts:** it changes how the app looks.

## 7. Open

- Upgrade Electron in 2.4.0? Recommended yes; needs a test on the Mac.
- Google Fonts or system fonts?
- Republish 2.4.0 to claude.ai?
- Does the owner write the skipped-rows count themselves?
- New repository or a branch (recommended: new repository), and its name.
- The app identifier.
- Does the family share one budget or keep separate ones?
- Which Mac (Apple Silicon or Intel)? Which Linux distribution?
- The per-phase decisions in `docs/PLAN.md` section 6; each has a recommendation, none is chosen yet.

## 8. Checklist

Phase 0, in this order (one writer in `app/index.html` at a time):
- [ ] Apply `budgeter-2.3.0.patch` and confirm it (Next step).
- [ ] Refuse to save over unreadable data (finding 1).
- [ ] Validate restored backups and escape the remaining values (finding 6).
- [ ] Add `appVersion` and the `check` block to the export; keep `version: 2`.
- [ ] Strict amount parsing and an "N rows couldn't be read" count (finding 5).
- [ ] Undo on transaction delete; ignore a blanked balance (finding 8).
- [ ] `crypto.randomUUID()` for new ids (finding 7).
- [ ] Automatic backup download before a restore (finding 2).
- [ ] The small items (finding 11) and https-only external links (finding 9).
- [ ] Three fixture backups (good, malformed, malicious) and a written manual checklist.
- [ ] Version 2.4.0 in `package.json` and `package-lock.json`; a 2.4.0 entry in `CHANGELOG.md`.
- [ ] Security review of the whole diff.
- [ ] Check every "Done when" item, with evidence.
- [ ] Give the owner "What you must understand" notes and the three Phase 0 questions from `docs/PLAN.md`.
- [ ] Hand the owner their tests: real-model sorting, `npm start` on Windows, the Mac.

Then the Opus rows of each phase in `docs/AGENT-PLAN.md` section 4:
- [ ] Phase 1: SQLCipher and keychain compile spike; review of capabilities and CSP.
- [ ] Phase 2: review that the page cannot reach the network.
- [ ] Phase 3: mentor the owner through `money.js` and the cents conversion.
- [ ] Phase 4: Rust database module, migration runner, apply-changes command with the allow-list, rotating backups; three schema checkpoints.
- [ ] Phase 5: keychain and open sequence, backup encryption, Linux fallback.
- [ ] Phase 6: single write path, hybrid logical clock, append-only guard.
- [ ] Phase 7: sync server, client and pairing.
- [ ] Phase 8: review of release workflows.

## 9. For Opus

- Before you act, open every file and connected app that could matter, including ones this brief doesn't name, and use what you find.
- Work through the checklist and tick each box as you go. Don't end with a summary that announces the next step: take it. Don't offer to continue: continue.
- Stop only for what's under "Ask me first", or when you need a decision only I can make.
- Don't reopen what's under "Decided". If something you find contradicts it, tell me.
- End with a short recap: what you did, and what's next.
