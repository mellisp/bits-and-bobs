# mpOS

Windows 2000/XP-themed personal website for matthewpritchard.com. Purely static (HTML/CSS/JS), no framework, no build step, no dependencies.

## Running locally

```
python3 -m http.server 8000
```

## Project structure

| File | Purpose |
|------|---------|
| `index.html` | Entire OS UI: all window divs, start menu, taskbar, desktop icons |
| `js/main.js` | All app logic (~8k lines): apps, settings, screen savers, desktop, voice commands |
| `js/taskbar.js` | Window management (open/close/minimize/restore, drag, resize, z-index), clock, system tray popups |
| `js/i18n.js` | i18n engine + English strings |
| `lang/pt.js` | Portuguese strings (registers via `_mpLangRegister`) |
| `js/audio.js` | Sound playback with cache |
| `css/theme.css` | XP design system (CSS vars, raised/sunken borders, animations) |
| `css/page.css` | Layout styles for index.html |
| `version.json` | App version (auto-bumped by pre-commit hook) |
| `brick-breaker.html`, `chicken-fingers.html`, `target-game.html` | Standalone game pages |
| `worker.js` | Cloudflare Worker for visitor map (deployed separately) |
| `scripts/build-fish-db.js` | Node script that builds `js/fish-data.js` from Wikipedia |

Data files lazy-loaded at runtime: `js/fish-data.js`, `js/aquarium-data.js`, `js/world-map-data.js`, `js/help-data.js`

## Architecture

- All JS files are IIFEs that export to `window`
- Script load order: `i18n.js` → `lang/pt.js` → `audio.js` → `taskbar.js` → `main.js` (all `defer`)
- Every app window is a pre-declared `<div>` in `index.html` (shown/hidden, never dynamically created)
- All state in `localStorage` (settings, language, volume, notepad files, icon positions)
- Inline `<head>` script reads settings before paint to prevent FOUC
- Pre-commit hook auto-bumps `version.json` patch number on every commit

## Key data structures (in `main.js`)

- **`FOLDER_ITEMS`** — `{ programs: [...], documents: [...], utilities: [...] }`. Each item has `name`, `_key` (i18n), `action` (e.g. `'openCalculator'`). Use `itemName(item)` for localized name.
- **`ACTION_MAP`** — Maps action strings to functions: `{ openCalculator: openCalculator, ... }`
- **`WINDOW_NAMES`** — Maps window div IDs to display names: `{ calculator: 'Calculator', ... }`
- **`COMMANDS`** — Terminal commands: `{ help: { run: fn, desc: '...' }, ... }`

## Cross-module API

| Export | Source | Purpose |
|--------|--------|---------|
| `window.mpTaskbar.{openWindow,closeWindow,minimizeWindow,restoreWindow,bringToFront}` | `taskbar.js` | Window management |
| `window.{t,tPlural,setLanguage,getLang}` | `i18n.js` | i18n |
| `window.mpVoiceStop` | `main.js` | Stop speech recognition externally |
| `window.mpAudioUpdateVolume` | `audio.js` | Sync volume after slider change |
| `openWindow(id)` | `main.js` | Show a window div + `bringToFront()` |

## Key patterns

- **i18n:** `data-i18n` attributes on elements, `t('key')` / `t('key', {var})` for lookups. `setLanguage()` scans all `[data-i18n]` and fires `languagechange` event. New strings need keys in both `js/i18n.js` and `lang/pt.js`.
- **Custom dialogs:** `mpConfirm()` returns a Promise — never use native `alert()`/`confirm()`/`prompt()`
- **Window IDs:** Derived from action names by stripping `open` prefix and lowercasing (e.g. `openCalculator` → `calculator`). The div ID in `index.html` matches this.
- **Adding an app:** Add to `FOLDER_ITEMS`, create an `open*` function calling `openWindow(id)`, add to `ACTION_MAP`, add window div to `index.html`, add `WINDOW_NAMES` entry, add i18n keys (`app.*.name`, `app.*.desc`, `win.*`).

## Conventions

- Win2000/XP aesthetic in all UI (beveled borders, system colors, Tahoma)
- Games are separate HTML pages, not embedded in index.html
- CSP meta tag in `index.html` — update when adding external sources

## Git commits

- **No Claude attribution** — never include `Co-Authored-By` lines or session URLs
- Keep commit messages concise and descriptive

## Deployment

- GitHub Pages from `main` branch
- Custom domain: `www.matthewpritchard.com` (via `CNAME`)
- Cloudflare Worker (`worker.js`) deployed separately for visitor tracking
