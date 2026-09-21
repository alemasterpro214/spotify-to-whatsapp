# spotify-to-whatsapp

Automatically updates your WhatsApp **About status ("thought")** and **classic profile description** (the "About" field on your contact card) with the song currently playing on **Spotify**.

> ⚠️ **Windows only** (reads the currently playing track through the Windows SMTC API, available on Windows 10/11 — including Tiny11 and LTSC 10 — and in "background" mode also on Windows 8.1/8/7/Vista, where Spotify track reading is not available to apps).


## Highlights

- **11 languages**: English, Italiano, 中文, Deutsch, Español, Русский, 日本語, हिन्दी, Français, 한국어, العربية. The **first time the app starts, it asks for your language before doing anything else** — with native names (中文, العربية, ...), so the prompt is understandable no matter which language you speak. The choice is remembered; the app is fully translated (logs, pairing screens, tray UI).
- **Two launch modes**:
  - `start.bat` — visible console with a **live HUD** (WhatsApp status, current track, log tail) on Windows 10/11 terminals;
  - `run-hidden.vbs` — **no terminal window at all**: a **tray icon** appears in the "Show hidden icons" area; click it to open the status window.
- **Start with Windows** button in the tray UI (asks for administrator permission once, so the entry also appears in Task Manager → Startup apps; removable from the app itself or from Task Manager).
- **Background running** option: when enabled, closing the window (or having no window at all) does NOT stop the app — it keeps running in the tray until you quit it from the tray UI ("Quit and restore").
- **Change the language at any time**: press **[K]** in the console window, or use the **Language** button in the tray window. The new language applies immediately (menu, logs) and is remembered.

## How it works

1. A small PowerShell script queries **Windows** and reads the currently playing track from Spotify (title, artist, album, state).
2. The program updates **two areas of your WhatsApp profile** whenever the song changes:
   - the **temporary status bubble** ("About" / thought);
   - the **classic profile description** ("About" field, max 139 characters, visible on your contact card) — can be enabled/disabled with `classicDescription`.
3. When Spotify stops playing (or when you exit the program), **both previous descriptions are automatically restored**, each back to its original text.
4. Everything happens **locally on your PC**: track reading does not send any data to remote servers.

## Privacy (Important)

- The **phone number** in `config.json` is **only** used to request the WhatsApp pairing code (QR-less login). It is never printed in logs: if it appears in an error message, it is sanitized as `[number removed]`.
- The **WhatsApp session** is saved locally in `.wwebjs_auth/` (project folder). No data is sent to third-party servers.
- The program **does not read chats, contacts, or messages**: it uses WhatsApp Web strictly to update your profile description/status.
- `config.json` and `.wwebjs_auth/` are included in `.gitignore`: they will never be committed or shared.
- The chosen language and tray UI state are stored locally (`.ui-language`) and are also gitignored.

## Installation

Requires [Node.js](https://nodejs.org) 18 or higher.

```bash
npm install
```

or run `installation.bat`.

## First launch

1. Copy `config.example.json` to `config.json` and enter your phone number (international format, digits only — see the table below).
2. Start the app:
   - double-click **`run-hidden.vbs`** for the hidden mode (recommended: no window, tray icon), or
   - run **`start.bat`** for the visible console (HUD).
3. **On the very first run the app asks for your language before anything else**:
   - in the console mode you type a number or a code (`en`, `it`, `zh`, ...);
   - in the hidden mode the same picker opens as a native window from the tray icon.
4. Then follow the WhatsApp pairing instructions shown in the window/console (pairing code or QR).

After initial pairing, your session remains saved: subsequent launches will not prompt for authentication.

## Configuration

```json
{
  "phone": "393401234567",
  "language": "",
  "pairingMode": "code",
  "statusTemplate": "🎵 {title} — {artist}",
  "idleStatus": "",
  "customDescriptionRestoring": "",
  "restoreOnExit": true,
  "classicDescription": true,
  "showAbout": true,
  "pollSeconds": 10,
  "appFilter": "Spotify",
  "backgroundRunning": false
}
```

| Field | Description |
|---|---|
| `phone` | Your phone number (including country code, digits only). Used only for the pairing code. |
| `language` | UI language: `en`, `it`, `zh`, `de`, `es`, `ru`, `ja`, `hi`, `fr`, `ko`, `ar`. Empty (default) = the choice made on first run (stored in `.ui-language`). This field has the highest priority. |
| `pairingMode` | `"code"` = login via 8-character pairing code (no QR scan needed); `"qr"` = classic QR code scan. |
| `statusTemplate` | Status text format. Placeholders: `{title}`, `{artist}`, `{album}`, `{app}`. |
| `idleStatus` | What to write when nothing is playing (Spotify closed or paused): `""` (empty, **default**) or `"restore"` = restore the description that was set before starting the program; `"none"` = leave the description untouched; any other text = set that text as the description (e.g. `"🎧 Away from streaming"`). |
| `customDescriptionRestoring` | **Priority override.** If set to a non-empty text, that text is written to BOTH profile fields (status bubble + classic "About" field) whenever nothing is playing AND when the program closes — replacing both `idleStatus` and the restore-on-exit behavior. Empty (default) = the field is ignored, as if it did not exist. The kebab-case alias `custom-description-restoring` is also accepted. |
| `restoreOnExit` | `true` (default) = upon exiting (Ctrl+C, tray Quit), restore pre-launch statuses. |
| `classicDescription` | `true` (default) = writes the song **also** to the classic profile description ("About" field, max 139 chars, on contact card), in addition to the status bubble. `false` = status bubble only. Does not touch status stories or chats. |
| `showAbout` | `true` (default) = displays your description in the terminal when saved/restored. `false` = hide it. |
| `pollSeconds` | Polling interval in seconds to check playing media (min 5, max 600). |
| `appFilter` | Only reads media sessions matching this string (`"Spotify"`). Empty `""` = any media app. |
| `backgroundRunning` | `false` (default) = closing the window quits the app (after restoring). `true` = the app keeps running when the window is closed; stop it from the tray icon ("Quit and restore"). This toggle can also be changed live from the tray UI. |

## The two launch modes

### Visible console (`start.bat`)

A live **HUD dashboard** shows: WhatsApp connection state, login mode, polling interval, filters, current track with playback state, last update time and the rolling log. It redraws in place using ANSI/VT sequences. On consoles that do not support them the HUD quietly disables itself and the output is the classic timestamped log — never garbage characters.

Keyboard shortcuts:
- **Ctrl+C** — stop the app (previous descriptions are restored first);
- **[K]** — open the language menu and switch language instantly.

While the [K] language menu is open the HUD pauses (so nothing is drawn over it); confirm your choice with Enter and the HUD comes back in the new language.

### Hidden mode (`run-hidden.vbs`)

No terminal window is ever opened. A **tray icon** (green WhatsApp-style dot with a music note) appears in the notification area ("Show hidden icons" — you can drag it out of the overflow to pin it). Clicking the icon opens the status window with:

- connection state and current track;
- **Start with Windows** — asks for administrator permission (UAC) once and registers the autostart entry in `HKLM\...\CurrentVersion\Run` with the hidden launcher. Entries in HKLM are exactly what **Task Manager → Startup apps** lists, so you can remove it from there, or just untick the checkbox in the app (which asks for permission again). Changing it never requires editing the registry by hand.
- **Background running** — keeps the app alive when the status window (or its whole host) is closed; the only way to stop it is the tray UI.
- **Language** — opens the language picker (same 11 languages); the new language applies immediately;
- **Quit and restore** — asks for confirmation, then restores your previous descriptions and exits completely (tray icon included).

The log files (`app.log`, `app.err`) are in the project folder and can be opened with any text editor.

Pairing, language choice and setup errors are shown as native windows in the selected language, so everything is usable without any console.

If the main process dies unexpectedly, the tray host detects it and closes itself (no orphaned icons).

## Windows compatibility

| Windows | Track reading (SMTC) | Console HUD | Hidden mode + tray UI |
|---|---|---|---|
| 11 / 10 / Tiny11 / LTSC 10 | ✅ | ✅ | ✅ |
| 8.1 / 8 / 7 / Vista | ❌ (system limitation) | ❌ (plain logs instead) | ✅ (app runs, tray UI works) |

On Windows 8.1/8/7/Vista the app starts normally and the tray UI works, but the system does not expose media information to apps, so no track can be detected (the log explains it). PowerShell is required for the media reader and the tray host (Windows PowerShell 2.0+ is sufficient: the tray host and the autostart helper avoid features newer than that).

## Testing

```bash
npm test            # unit tests (config, formatting, privacy, i18n, HUD, tray codec)
npm run test:media # actually reads Windows media sessions
npm run test:whatsapp
```

`test:media` displays detected media sessions and the status that would be set: **open Spotify and start playing a track** before running it. `test:whatsapp` verifies the WhatsApp Web connection without altering your profile (exits with code 2 if you do not complete pairing within the timeout, which is expected behavior).

## Description Visibility ("Empty About" Fix)

Since late 2025, WhatsApp converted the About section into a **temporary status**: each status contains text (max 50 characters), an optional emoji, and a **duration** (1h, 8h, 1d, 2d, 1 week). Mobile apps display the status bubble **only if a valid duration is set**: without a duration, the text remains stored on the server but **never appears** (the root cause of the classic "empty About" bug).

Legacy methods (`client.setStatus` from the library and the old IQ `sendSetAbout`) update the legacy field **without duration**, which is why the text failed to appear on phones.

This program uses the **GraphQL mutation** `WAWebMexUpdateTextStatusJob.mexUpdateTextStatus`, the same internal pathway used by WhatsApp Web and mobile apps: it sets **text, emoji, and duration (7 days)** in a single request, ensuring the bubble is **visible in the mobile app**. Reading the description also uses the new GraphQL fetch approach, ensuring that restoration accurately captures the exact text displayed on phones.

In addition:
- if a song remains static for hours, the description is **automatically re-asserted every 12 hours** so it never expires while the application is running;
- if the template exceeds the 50-character limit of the new About status, the text is **automatically truncated**;
- with `classicDescription: true`, the **same text** is also written to the classic "About" field (truncated to 139 characters): the status bubble and classic field are **two distinct server-side fields** and are updated/restored independently.

> Note: Application logs will alert you if a status was set using a fallback method (in which case it might not appear on mobile devices).

## Troubleshooting

- **"No active media session"** → Spotify must be playing (not paused) and visible in the Windows Media Control panel.
- **Pairing code doesn't work** → Codes expire after a few minutes; restart the app to generate a new code. Ensure `phone` is correctly formatted with country code.
- **"Linked account DOES NOT match"** → The phone number in `config.json` does not match the linked account. Correct `config.json`, or delete the `.wwebjs_auth` folder and re-pair.
- **Chromium doesn't launch** → The initial run downloads Chromium (may take a few minutes).
- **"The browser is already running"** → Another instance of the script is already running or hung. Close it before relaunching.
- **The description stayed on the song after closing** → The process was killed before the restore finished. Just start the app again: it completes the restore automatically (auto-repair), or run `node scripts/restore-about.js "your text"` to set it manually.
- **Quick manual test** → `node scripts/test-set-about.js` sets a test status, verifies it, and restores the previous description.
- **About Diagnostics** → `scripts/diag-about*.js`: inspect internal WhatsApp Web modules and test write pathways, verifying text, emoji, and duration server-side. `diag-about11.js` is the key script (GraphQL mutation).
- **Wrong language** → set `"language": "en"` (or any other code) in `config.json`, or delete the `.ui-language` file to be asked again on the next start.
- **The tray icon did not appear** → PowerShell must be available (it is on every Windows installation). The app keeps running hidden anyway; check `app.err` for details.

## Technical Notes

- Track reading uses `GlobalSystemMediaTransportControlsSessionManager` (SMTC) via PowerShell 5.1-compatible syntax: no native binary compilation required.
- The tray host is a WinForms `NotifyIcon` in a hidden PowerShell process; Node ↔ tray communication happens through two small local files with URL-encoded key=value lines (PowerShell 2.0-compatible, no JSON dependency).
- Description updates rely on [`whatsapp-web.js`](https://wwebjs.dev) (`client.setStatus`) driving WhatsApp Web. This is an unofficial tool: use responsibly (by default, updates occur only when the track changes, not on every poll interval).
- The classic "About" profile field is written via the internal `WAWebSetAboutJob` (legacy IQ, max 139 characters), with fallback to `client.setStatus`: this is **the exact field** shown on your contact card, distinct from the new timed status bubble. The program **never** touches status updates/stories or chats.
- Translations live in `src/locales/*.js`; every locale is checked against the English key set by the unit tests, so a missing string can never produce broken output.

## Sharing a Clean Copy

To share the app with someone as if it had never been used, select and zip **only** these items:

```
spotify-to-whatsapp/
├── src/                  (all files, including locales/)
├── scripts/              (all files)
├── test/                 (all files)
├── node_modules/         (optional: can be regenerated with `npm install`)
├── .gitignore
├── config.example.json
├── installation.bat
├── package.json
├── package-lock.json
├── README.md
├── run-hidden.vbs
└── start.bat
```

**Do NOT include** these (they contain personal data or are machine-specific):

- `config.json` — contains the real phone number;
- `.wwebjs_auth/` — contains the WhatsApp session bound to your account (whoever has it could use your WhatsApp);
- `.wwebjs_cache/` — WhatsApp Web cache;
- `restore-state.json` — pending-restore state (profile description texts);
- `.ui-language` — language preference (trivial, but the recipient should choose their own);
- `.tray-status`, `.tray-command` — tray runtime files;
- `app.log`, `app.err`, `app.lock`, `smoke2.log` — runtime logs and lock file.
The recipient copies `config.example.json` to `config.json`, enters their own phone number, and runs `npm install` (or `installation.bat`) if `node_modules/` was not included.
