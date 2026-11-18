# Budgeter desktop: master agent plan

Written 2026-11-16. This says which parts of [PLAN.md](PLAN.md) go to which agent and model. PLAN.md stays the source for what each phase is and when it is done; this file only covers who does it.

## 1. How the work is divided

| Who | Model | Gets work that is... |
|---|---|---|
| Lead session | Opus 5.5 (`claude-opus-5-5`) | Holding the plan, talking to the owner, splitting tasks, checking every report before passing it on |
| Opus agents | Opus 5.5 | Costly if subtly wrong: encryption, the database write path, merge and sync, security review. Also teaching and design review |
| Sonnet agents | Sonnet 5.5 (`claude-sonnet-5-5`) | Building to a clear spec with a written "Done when" check |
| Haiku agents | Haiku 5.5 (`claude-haiku-5-5`) | Mechanical: the instruction fully determines the result. Also read-only pass/fail checks |
| The owner | — | The parts they are learning by writing: schema, money handling, `diff()`, conflict rules, the sync protocol, front-end changes from Phase 3, tests, the threat model |

The lead starts from [OPUS-BRIEF.md](OPUS-BRIEF.md).

## 2. The agents

Each is a file in `.claude/agents/`, with its model set in the file. In Claude Code, name the agent in the request ("use the mentor agent to review my schema").

| Agent | Model | Does | Never does |
|---|---|---|---|
| `mentor` | Opus | Coaches the owner on their parts: concepts, hints, checkpoints, reviews of their code and tests | Writes the owner's finished code; edits files |
| `rust-core` | Opus | Security-critical Rust: SQLCipher and keychain, backup encryption, the single write path, migrations runner, change log, logical clock, sync server and pairing | Writes the schema, merge SQL or conflict rules |
| `security-reviewer` | Opus | Read-only review of a diff or phase against the trust boundaries | Edits files |
| `tauri-builder` | Sonnet | Tauri shell, config, capabilities, platform adapter, Ollama bridge commands, CI, packaging, release docs drafts | Touches keys, the write path or sync authentication |
| `frontend-engineer` | Sonnet | Specified front-end changes: Phase 0 fixes, pdf.js in the page, the move-only module split | Writes the owner's front-end parts from Phase 3 on |
| `test-harness` | Sonnet | Test scaffolding: runner setup, golden harness, fake Ollama, migration scaffolding, multi-device simulation, one worked example each | Writes the owner's test cases |
| `repo-clerk` | Haiku | Exact edits: apply a named patch, bump versions, changelog entries, README fixes, fixture files from a spec | Makes judgment calls |
| `acceptance-checker` | Haiku | Runs tests and greps, compares "Done when" lists to the repo, reports pass or fail with evidence | Fixes anything |

## 3. Rules for every agent

1. **Read first:** `CLAUDE.md`, `docs/PLAN.md` and this file.
2. **The owner's parts stay the owner's.** For those, agents give hints, checkpoints and reviews, not finished code, unless the owner asks outright.
3. **One writer per file at a time.** Phase 0 is all in `app/index.html`, so its code tasks run one after another, never side by side.
4. **Ask the owner before:** committing, pushing, publishing, installing or upgrading software, deleting files, creating a repository, or anything under "Decided" in PLAN.md changing.
5. **AI-written code arrives with** "What you must understand" notes and three questions the owner should be able to answer. Rust is written for a beginner to read.
6. **Reports say** what changed, how it was checked (the command and what it printed), and what is not done. The lead checks the result itself before telling the owner it is done.
7. **Review gate:** anything touching a trust boundary goes through `security-reviewer` before it is accepted, and every phase ends with `security-reviewer` then `acceptance-checker`.
8. **Escalate up, not sideways:** Haiku agents stop at any judgment call; Sonnet agents hand anything involving keys, the write path or sync authentication to `rust-core`; anything that contradicts a decision goes to the owner.

## 4. Delegation by phase

Tasks within a phase are listed in the order they should run.

### Phase 0: Budgeter 2.4.0 stopgap

| Task | Agent | Model |
|---|---|---|
| Apply `budgeter-2.3.0.patch` (it is cumulative; skip its `CHANGELOG.md` part) | `repo-clerk` | Haiku |
| Refuse to save over unreadable data (finding 1) | Lead | Opus |
| Validate restored backups; escape the remaining values (finding 6) | Lead | Opus |
| Add `appVersion` and the `check` block to the export | Lead | Opus |
| Strict amount parsing and the skipped-rows count (5); undo on delete and blanked balance (8); `crypto.randomUUID()` (7); backup before restore (2); small items (11); https-only links (9) | `frontend-engineer` | Sonnet |
| Electron upgrade, if the owner approves | `frontend-engineer` | Sonnet |
| Three fixture backups from the lead's spec; manual checklist; version 2.4.0; changelog entry; README `cd` fix | `repo-clerk` | Haiku |
| Review the whole diff | `security-reviewer` | Opus |
| Check the "Done when" list | `acceptance-checker` | Haiku |
| Real-model sorting test; run on Windows and on the Mac | Owner | — |

The owner may choose to write the skipped-rows count themselves, with `mentor`.

### Phase 1: Tauri shell

| Task | Agent | Model |
|---|---|---|
| Rust shell, `tauri.conf.json`, capabilities, `platform.js`, CI build on three systems | `tauri-builder` | Sonnet |
| Spike: SQLCipher and the keychain library compile on all three | `rust-core` | Opus |
| ~~Bundle the fonts~~ Not needed: 2.4.0 uses system fonts | — | — |
| Move the inline script and styles into files; smoke checklist | Owner, with `mentor` | Opus |
| Review capabilities and CSP | `security-reviewer` | Opus |
| Check "Done when" | `acceptance-checker` | Haiku |

### Phase 2: Local AI

| Task | Agent | Model |
|---|---|---|
| Rust commands for status, JSON answers, streamed chat and stop | `tauri-builder` | Sonnet |
| Bridge script exposing `window.budgeterDesktop`; pdf.js in the page | `frontend-engineer` | Sonnet |
| Fake Ollama server and Rust tests against it | `test-harness` | Sonnet |
| Review: page cannot reach the network; model output stays untrusted | `security-reviewer` | Opus |
| Accuracy log on 50 merchants | Owner | — |
| Check "Done when" | `acceptance-checker` | Haiku |

### Phase 3: Modules, first tests, money in cents

| Task | Agent | Model |
|---|---|---|
| Test runner setup, `npm test`, CI job, golden harness | `test-harness` | Sonnet |
| Move-only split into modules | `frontend-engineer` | Sonnet |
| Confirm the split changed no behaviour (golden test, diff check) | `acceptance-checker` | Haiku |
| `money.js` test first; cents conversion area by area; renames; backup importer | Owner, with `mentor` | Opus |
| Grep audit: no `parseFloat` or `toFixed` outside the money and parsing modules | `acceptance-checker` | Haiku |

### Phase 4: SQLite

| Task | Agent | Model |
|---|---|---|
| Schema (three checkpoints), `diff()` with tests, backup-to-rows import | Owner, with `mentor` | Opus |
| Rust database module, migration runner, apply-changes command with the allow-list, rotating backups | `rust-core` | Opus |
| Migration test scaffolding and fixture loaders | `test-harness` | Sonnet |
| Review: parameterized SQL and the allow-list | `security-reviewer` | Opus |
| Integrity checks and "Done when" | `acceptance-checker` | Haiku |

### Phase 5: Encryption

| Task | Agent | Model |
|---|---|---|
| Keychain and open sequence, backup encryption, Linux fallback | `rust-core` | Opus |
| Threat model one-pager; test matrix | Owner, with `mentor` | Opus |
| Review key handling | `security-reviewer` | Opus |
| Search database files for a known merchant name; "Done when" | `acceptance-checker` | Haiku |

### Phase 6: Change log, history, undo

| Task | Agent | Model |
|---|---|---|
| `changes` schema, apply-and-inverse SQL, history view, scenario tests | Owner, with `mentor` | Opus |
| Single write path, hybrid logical clock, append-only guard | `rust-core` | Opus |
| Rebuild-test and undo-sequence helpers | `test-harness` | Sonnet |
| Review log completeness | `security-reviewer` | Opus |
| Run the rebuild test; "Done when" | `acceptance-checker` | Haiku |

### Phase 7: Sync over Tailscale

| Task | Agent | Model |
|---|---|---|
| Protocol document, conflict rules table, merge SQL, conflicts screen, scenario tests | Owner, with `mentor` | Opus |
| Server, client, pairing, listening only on the Tailscale address | `rust-core` | Opus |
| Multi-device simulation helpers | `test-harness` | Sonnet |
| Review authentication, binding, size limits | `security-reviewer` | Opus |
| Convergence runs; "Done when" | `acceptance-checker` | Haiku |

### Phase 8: Releases

| Task | Agent | Model |
|---|---|---|
| Release workflows, checksums, release notes template, install docs draft | `tauri-builder` | Sonnet |
| Review workflow permissions and pinned actions | `security-reviewer` | Opus |
| Changelog and version bookkeeping | `repo-clerk` | Haiku |
| Verify checksums; "Done when" | `acceptance-checker` | Haiku |
| Release checklist; final install docs; fresh-install tests | Owner | — |

## 5. Share of the work by model

- **Opus:** all of the security-critical Rust in Phases 4 to 7, every review, all mentoring, and the three trust-boundary fixes in Phase 0.
- **Sonnet:** the Tauri shell, the AI bridge, CI and releases, test scaffolding, and the specified front-end work in Phases 0 to 3.
- **Haiku:** patches, versions, changelogs, fixtures, and every end-of-task check.
- **Owner:** the schema, money, `diff()`, conflict rules, protocol, tests, and the front end from Phase 3 on.
