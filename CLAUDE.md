# Budgeter

Monthly budget planner. The whole app is one file, `app/index.html` (HTML, CSS and vanilla JS, no build step). It runs in two places:

- **A browser**, by opening `app/index.html`. Data is in that browser's localStorage only.
- **The desktop app** (Electron, `main.js`; built for the Mac by `install.sh`, `npm start` elsewhere).

**This fork is not hosted on claude.ai.** The original Budgeter project was published as a claude.ai artifact, so the page still has code paths for `window.claude` (account sync through `claude.use('db')`, `downloads`, `sample`, `user`). They stay dormant here because `window.claude` never exists. Don't publish this app to claude.ai, and don't plan work around it.

Desktop details: `window.claude` doesn't exist there, so the app falls back to localStorage only and saves files with an `<a download>` link. AI features (reading PDFs, screenshots and statement text; Ask about your spending) use a local model, `gemma3:4b`, through Ollama on 127.0.0.1:11434. The page reaches it through `window.budgeterDesktop` (`preload.js`); `main.js` makes the Ollama calls (JSON-schema output for transactions, streamed chat for questions) and pulls PDF text with `pdfjs-dist`. In the page, `sampleCaps()` hides the difference between Claude on the web and Ollama on desktop: `s.json()` reads transactions and `chat()` answers questions.

This folder is the source of truth. Edit `app/index.html` here; the desktop app picks it up on the next `npm start` or `bash install.sh`.

## Rules for changes
- Every `claude.use(...)` call must keep working when it resolves `null` or `window.claude` is missing. That is always the case in this fork.
- The page loads nothing from the internet: system fonts only, no CDN scripts or stylesheets. Only `main.js` talks to the network, and only to Ollama on this machine.
- Don't break saved data: the localStorage prefix is `budgeter:`, backups are `{app:'budgeter',version:2,plan,tx}`, and restore also accepts the old `monthly-planner` v1 backups. Put schema changes in `upgrade(p)`, which runs on load and on restore.
- Keep the light/dark tokens on `:root` and the phone layout (≤640px) working.
- Don't trust a small model with signs or arithmetic: the AI copies amounts as printed and the app sets the sign (`fromClaude`); for questions the app computes every total in `spendingFacts()` and the AI only explains them.

## Run
- Browser preview: the `budgeter` config in `.claude/launch.json` serves `app/` on port 8765 (localStorage-only mode, same as desktop).
- Desktop dev: `npm install` once, then `npm start`. Install or update the Mac app: `bash install.sh` (builds with electron-builder and copies to /Applications). Electron 44 and later need macOS 13 or newer.

## Machines
This project is developed on more than one computer (Windows, macOS, Linux).
- Committed files use paths relative to the repo root. They never name a machine's folders, user, or installed tool versions.
- Each machine's own settings are in `.env` at the repo root, which is not committed. `.env.example` lists the keys. If `.env` is missing, copy `.env.example` to `.env` and fill it in for that machine.
- Don't assume an operating system or shell: read `DEV_OS` in `.env`. Find tool versions and Ollama models by running the tools (`node -v`, `rustc --version`, `ollama list`), not from notes.
- `.gitattributes` keeps line endings consistent across machines. Don't convert them by hand.

## History
Monthly planner is the v1 ancestor. The first git commit is the old desktop build of it. Budgeter 2.0 began as a claude.ai artifact in the original project; this fork (from 2.1.0) is desktop and browser only.
