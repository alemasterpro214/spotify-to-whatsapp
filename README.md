# spotify-whatsapp-status

Automatically updates your WhatsApp **About status ("thought")** and **classic profile description** (the "About" field on your contact card) with the song currently playing on **Spotify**.

> ⚠️ **Windows 10/11 only** (uses Windows SMTC API to read the currently playing track).

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

## Installation

Requires [Node.js](https://nodejs.org) 18 or higher.

```bash
npm install
```

## Configuration

1. Copy `config.example.json` to `config.json`
2. Enter your phone number in international format (digits only, no `+` or spaces):

```json
{
  "phone": "393401234567",
  "pairingMode": "code",
  "statusTemplate": "🎵 {title} — {artist}",
  "idleStatus": "restore",
  "restoreOnExit": true,
  "pollSeconds": 10,
  "appFilter": "Spotify"
}
```

| Field | Description |
|---|---|
| `phone` | Your phone number (including country code, digits only). Used only for the pairing code. |
| `pairingMode` | `"code"` = login via 8-character pairing code (no QR scan needed); `"qr"` = classic QR code scan. |
| `statusTemplate` | Status text format. Placeholders: `{title}`, `{artist}`, `{album}`, `{app}`. |
| `idleStatus` | Action when nothing is playing: `"restore"` = revert to the status active before launching (recommended), any custom string = set custom text, `""` = leave status untouched. |
| `restoreOnExit` | `true` (default) = upon exiting (Ctrl+C), restore pre-launch statuses. |
| `classicDescription` | `true` (default) = writes the song **also** to the classic profile description ("About" field, max 139 chars, on contact card), in addition to the status bubble. `false` = status bubble only. Does not touch status stories or chats. |
| `showAbout` | `true` (default) = displays your description in the terminal when saved/restored. `false` = hide it. |
| `pollSeconds` | Polling interval in seconds to check playing media (min 5, max 600). |
| `appFilter` | Only reads media sessions matching this string (`"Spotify"`). Empty `""` = any media app. |

## Usage

```bash
npm start
```

On first run:

- **`code` mode**: an 8-character code like `ABCD-EFGH` is displayed. On your phone, open **WhatsApp → Settings → Linked devices → Link a device → "Link with phone number instead"** and enter the code.
- **`qr` mode**: scan the QR code following the standard procedure.

After initial pairing, your session remains saved: subsequent launches will not prompt for authentication.

When Spotify is paused or closed, your status **automatically reverts to what you had before starting the program** (when using `idleStatus: "restore"`). The same occurs when you terminate the script with `Ctrl+C` (thanks to `restoreOnExit`).

To stop the program: `Ctrl+C`.

## Testing

```bash
npm test            # unit tests (config, formatting, privacy)
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
- **Quick manual test** → `node scripts/test-set-about.js` sets a test status, verifies it, and restores the previous description.
- **About Diagnostics** → `scripts/diag-about*.js`: inspect internal WhatsApp Web modules and test write pathways, verifying text, emoji, and duration server-side. `diag-about11.js` is the key script (GraphQL mutation).

## Technical Notes

- Track reading uses `GlobalSystemMediaTransportControlsSessionManager` (SMTC) via PowerShell 5.1: no native binary compilation required.
- Description updates rely on [`whatsapp-web.js`](https://wwebjs.dev) (`client.setStatus`) driving WhatsApp Web. This is an unofficial tool: use responsibly (by default, updates occur only when the track changes, not on every poll interval).
- The classic "About" profile field is written via the internal `WAWebSetAboutJob` (legacy IQ, max 139 characters), with fallback to `client.setStatus`: this is **the exact field** shown on your contact card, distinct from the new timed status bubble. The program **never** touches status updates/stories or chats.

For now (version 1.0), the app is only available in Italian, and for PCs.
