# mpOS

Retro desktop-themed personal website for matthewpritchard.com. Purely static (HTML/CSS/JS), no framework, no build step, no dependencies.

## Running locally

```
python3 -m http.server 8000
```

## Project structure

| File | Purpose |
|------|---------|
| `index.html` | Entire OS UI: all window divs, start menu, taskbar, desktop icons |
| `js/core.js` | Shared helpers, registration hooks, window management |
| `js/explorer.js` | File browser (FOLDER_ITEMS, navigation) |
| `js/system.js` | System Properties (4 tabs), screensaver, display settings |
| `js/notepad.js` | Notepad (save/load/find/replace) |
| `js/calculator.js` | Calculator (basic + scientific) |
| `js/paint.js` | Paint (canvas, undo/redo, tools) |
| `js/terminal.js` | Terminal (60+ commands, virtual filesystem) |
| `js/calendar.js` | Calendar + Time Zone |
| `js/weather.js` | Weather (Open-Meteo API) |
| `js/games.js` | Game launchers (OnTarget, BrickBreaker, ChickenFingers, Aquarium, Fractal) |
| `js/fish.js` | Fish of the Day + Fish Finder |
| `js/browser.js` | WikiBrowser + Archive Browser |
| `js/media.js` | Noise Mixer + Tuning Fork |
| `js/utilities.js` | Stopwatch, Sticky Notes, Cryptography, Disk Usage, Slot Machine |
| `js/data-apps.js` | NEO Tracker, Visitor Map, Task Manager |
| `js/search.js` | Search + Help system |
| `js/desktop.js` | Bootstrap: desktop icons, voice commands, context menu, init |
| `js/taskbar.js` | Window management (open/close/minimize/restore, drag, resize, z-index), clock, system tray |
| `js/i18n.js` | i18n engine + English strings |
| `lang/pt.js` | Portuguese strings (registers via `_mpLangRegister`) |
| `js/audio.js` | Sound playback with cache |
| `css/theme.css` | Retro design system (CSS vars, raised/sunken borders, animations) |
| `css/page.css` | Layout styles for index.html |
| `version.json` | App version + disk usage (auto-bumped by pre-commit hook) |
| `brick-breaker.html`, `chicken-fingers.html`, `target-game.html` | Standalone game pages |
| `worker.js` | Cloudflare Worker for visitor map (deployed separately) |
| `scripts/build-fish-db.js` | Node script that builds `js/fish-data.js` from Wikipedia |

Data files lazy-loaded at runtime: `js/fish-data.js`, `js/aquarium-data.js`, `js/world-map-data.js`, `js/help-data.js`

## Architecture

- All JS files are IIFEs that export to `window`
- Script load order (all `defer`): `i18n.js` → `lang/pt.js` → `audio.js` → `taskbar.js` → `core.js` → `explorer.js` → `system.js` → `notepad.js` → `calculator.js` → `paint.js` → `terminal.js` → `calendar.js` → `weather.js` → `games.js` → `fish.js` → `browser.js` → `media.js` → `utilities.js` → `data-apps.js` → `search.js` → `desktop.js`
- Each app file self-registers via `mpRegisterActions()`, `mpRegisterWindows()`, `mpRegisterCommands()`
- Every app window is a pre-declared `<div>` in `index.html` (shown/hidden, never dynamically created)
- All state in `localStorage` (settings, language, volume, notepad files, icon positions)
- Inline `<head>` script reads settings before paint to prevent FOUC
- Pre-commit hook auto-bumps `version.json` patch number on every commit

## Key data structures

- **`FOLDER_ITEMS`** (explorer.js) — `{ programs: [...], documents: [...], utilities: [...] }`. Each item has `name`, `_key` (i18n), `action` (e.g. `'openCalculator'`). Use `itemName(item)` for localized name.
- **`ACTION_MAP`** (core.js, populated by each app) — Maps action strings to functions: `{ openCalculator: openCalculator, ... }`
- **`WINDOW_NAMES`** (core.js, populated by each app) — Maps window div IDs to display names: `{ calculator: 'Calculator', ... }`
- **`COMMANDS`** (core.js, populated by terminal.js) — Terminal commands: `{ help: { run: fn, desc: '...' }, ... }`
- **`CLOSE_MAP`** (core.js, populated by apps with custom close logic) — Maps window IDs to close handlers

## Cross-module API

| Export | Source | Purpose |
|--------|--------|---------|
| `window.mpTaskbar.{closeWindow,minimizeWindow,restoreWindow,bringToFront}` | `taskbar.js` | Window management |
| `window.{t,tPlural,setLanguage,getLang}` | `i18n.js` | i18n |
| `window.{openWindow,mpConfirm,getLocation,getItemIcon,loadDataScript}` | `core.js` | Shared helpers |
| `window.{mpRegisterActions,mpRegisterWindows,mpRegisterCommands}` | `core.js` | App self-registration |
| `window.mpVoiceStop` | `desktop.js` | Stop speech recognition externally |
| `window.mpAudioUpdateVolume` | `audio.js` | Sync volume after slider change |
| `window.notepadOpenWithContent(name, content)` | `notepad.js` | Open notepad with specific content |
| `window.displaySettings` | `system.js` | Shared display settings object |
| `window.FOLDER_ITEMS`, `window.FOLDER_NAMES` | `explorer.js` | Folder data for search, terminal, voice |
| `window.FILESYSTEM` | `terminal.js` | Virtual filesystem for search |

## Key patterns

- **i18n:** `data-i18n` attributes on elements, `t('key')` / `t('key', {var})` for lookups. `setLanguage()` scans all `[data-i18n]` and fires `languagechange` event. New strings need keys in both `js/i18n.js` and `lang/pt.js`.
- **Custom dialogs:** `mpConfirm()` returns a Promise — never use native `alert()`/`confirm()`/`prompt()`
- **Window IDs:** Derived from action names by stripping `open` prefix and lowercasing (e.g. `openCalculator` → `calculator`). The div ID in `index.html` matches this.
- **Adding an app:** Create a new JS file (or add to an existing one). Call `mpRegisterActions({ openMyApp })` and `mpRegisterWindows({ myapp: 'My App' })`. Add `window.openMyApp = openMyApp` for inline handlers. Add to `FOLDER_ITEMS` in `explorer.js`. Add window div to `index.html`. Add i18n keys (`app.*.name`, `app.*.desc`, `win.*`). Add `<script defer>` tag to `index.html` in the correct load position.
- **Language refresh:** Each module exports a `*RefreshOnLangChange()` function called by `desktop.js`'s `languagechange` handler.

## Conventions

- Retro desktop aesthetic in all UI (beveled borders, system colors, classic fonts)
- Games are separate HTML pages, not embedded in index.html
- CSP meta tag in `index.html` — update when adding external sources
- Modern JS syntax: `const`/`let`, arrow functions, `async`/`await`, template literals, optional chaining, destructuring

## Git commits

- **No Claude attribution** — never include `Co-Authored-By` lines or session URLs
- Keep commit messages concise and descriptive
- **Always `git pull` before committing** — multiple Claude windows may be editing concurrently

## Deployment

- GitHub Pages from `main` branch
- Custom domain: `www.matthewpritchard.com` (via `CNAME`)
- Cloudflare Worker (`worker.js`) deployed separately for visitor tracking
