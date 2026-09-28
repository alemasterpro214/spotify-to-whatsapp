# spotify-to-whatsapp

Automatically shows **the song you are listening to on Spotify** in your
**WhatsApp status (About)** and, optionally, in your **classic profile
description** — in two builds that do the same thing on two different systems:

| | **Windows** (`for pc/`) | **Android** (`for android/`) |
|---|---|---|
| Language / stack | Node.js 18+ (JavaScript) | Kotlin (native, no runtime deps) |
| Ship as | `spotify-to-whatsapp.zip` — ready for GitHub | `spotify-to-whatsapp-android.apk` — ready to install |
| Reads the track from | Windows SMTC API (PowerShell) | Android media session (`NotificationListenerService`) |
| Talks to WhatsApp via | WhatsApp Web driven by `whatsapp-web.js` (Chromium) | WhatsApp Web hosted in an in-app WebView |
| Pairing code (8 digits) | ✅ | ✅ (WhatsApp's own "Link with phone number" screen) |
| Start with the system | ✅ Task Manager → Startup apps | ✅ Starts at phone boot |
| Sensitive data encrypted at rest | ✅ Phone number + WhatsApp session (DPAPI + AES-256-GCM) | ✅ Settings + restore state (Android Keystore + AES-256-GCM) |
| ADB / root / special access required | ❌ | ❌ |
| UI languages | 11 (console + tray UI) | English + Italiano (follows the phone) |
| Background running | Tray icon / hidden mode | Foreground service (`specialUse`) + boot receiver |

Both builds share the same idea and the same WhatsApp internals, and both restore
your previous description when playback stops or when you quit.

---

## 1. Repository layout

```
spotify-to-whatsapp-developing/
├── README.md                        ← this file
├── for pc/                          Windows build (Node.js)
│   ├── spotify-to-whatsapp.zip      clean, shareable, GitHub-ready archive
│   ├── src/                         source (config, formatting, encryption, UI, WhatsApp)
│   ├── scripts/                     helpers (media reader, tray host, DPAPI, vault CLI, smoke tests)
│   ├── test/                        unit tests (89)
│   ├── config.example.json
│   ├── installation.bat             one-click setup
│   ├── start.bat                    visible console (HUD)
│   └── run-hidden.vbs               no window, tray icon
└── for android/                     Android build (Kotlin)
    ├── spotify-to-whatsapp-android.apk   ready to install
    ├── app/                         the Android project
    ├── build-apk.sh                 one command to rebuild the APK
    └── README.md                    Android-specific documentation
```

The two folders are independent: you can copy just one of them.

---

## 2. What it does

1. It reads the track that **Spotify** (or any media app you point it at) is
   currently playing — title, artist, album, playback state.
2. It formats it with your template, by default `🎵 {title} — {artist}`.
3. It writes the result to **two independent fields of your own WhatsApp
   profile**:
   - the **timed status** ("About" bubble), max 50 characters, written with a
     7-day duration so it actually appears on phones;
   - the **classic profile description** ("About" on your contact card), max
     139 characters — can be switched off.
4. When Spotify stops (or when you quit the app), **both previous values are put
   back** exactly as they were, each into its own field.
5. While it runs, the status is **re-asserted every 12 hours** so it never
   expires, and it is re-written on every track change.

It never touches status stories, chats, contacts or messages.

---

## 3. Privacy and encryption (both builds)

This was a hard requirement, and it is implemented in both versions.

### Nothing personal leaves the machine/phone

- The only outbound network traffic is the one WhatsApp itself needs. No
  analytics, no telemetry, no crash reporting, no third-party SDK, no QR relay.
- On Android, cleartext HTTP is refused at platform level; the APK has **zero
  runtime dependencies**.
- On Windows the browser used for WhatsApp Web is launched by explicit path, and
  the app has no dependency that phones home.
- Logs never contain your number: any run of 6+ digits is rewritten as
  `[number removed]`. On Android the log is **in memory only** — nothing is
  written to disk at all.

### Encrypted at rest

| | Windows | Android |
|---|---|---|
| Cipher | AES-256-GCM (authenticated) | AES-256-GCM (authenticated) |
| Key | 256-bit master key in `.secrets/master.key`, wrapped with the **Windows Data Protection API** (DPAPI, current user) so it can only be unwrapped by *your* Windows account on *your* machine | 256-bit AES key generated **inside the Android Keystore**, non-exportable (hardware-backed where the device supports it) |
| Phone number | stored as `"phone": "enc:v1:…"` in `config.json` after the first run | inside the encrypted settings blob |
| WhatsApp session | `.wwebjs_auth` is stored as `<name>.enc` and decrypted only while the app runs; re-encrypted on exit, with auto-repair if the process is killed | kept in the app's **private** storage; cloud backup and device transfer are disabled |
| Backup | – | `allowBackup="false"` + `data_extraction_rules.xml` exclude every domain |

**Honest limits** (documented in both READMEs): while an app is *running* the
session has to be readable — encryption protects the data **at rest** (files you
copy, sync, back up or lose), not the memory of a process that malware already
controls. On Android, Chromium's own WebView storage is not individually
encrypted; it is protected by the app sandbox and by the disabled backup.

---

## 4. Windows version — quick start

```text
1. Install Node.js 18+  (https://nodejs.org)
2. Run  installation.bat          (checks Node, installs a browser if needed, installs deps)
3. Run  npm run make-zip          → spotify-to-whatsapp.zip  (optional, for sharing)
4. Copy  config.example.json  →  config.json  and put your phone number in "phone"
5. Double-click  run-hidden.vbs   (tray icon, no window)   or  start.bat  (visible console)
6. Follow the pairing instructions (8-digit code or QR)
```

The first launch asks for your language before doing anything else (11 languages,
shown with their native names). After pairing, the session is remembered.

Full documentation, all configuration fields, the HUD/tray UI and the Windows
compatibility matrix are in **[`for pc/README.md`](for%20pc/README.md)**.

### The shareable archive

`for pc/spotify-to-whatsapp.zip` is generated by `npm run make-zip` and contains
**only** what a new user needs. The generator *refuses* to include anything
private, and the following are hard-coded as forbidden:

`config.json` (the phone number) · `.wwebjs_auth` and `*.enc` (the WhatsApp
session) · `.secrets` (the encryption key) · `.wwebjs_cache` ·
`restore-state.json` · `.ui-language` · `.tray-*` · `app.log` · `app.err` ·
`app.lock` · `node_modules` · `chromium`

So the zip is safe to attach to a chat or to upload to GitHub as-is. The
recipient runs `installation.bat`, copies `config.example.json` to `config.json`
and types their own number.

---

## 5. Android version — quick start

```text
1. Copy  for android/spotify-to-whatsapp-android.apk  to the phone and tap it
2. Open the app; it shows four steps, one button each:
     1. Open notification access      (Settings → Notification access)
     2. Open WhatsApp linking         (WhatsApp's own page, inside the app —
                                       use "Link with phone number instead", or the QR)
     3. Start at boot                 (already automatic on most phones)
     4. Allow background running      (system dialog)
3. Press Start.
```

On Xiaomi/Redmi/POCO (MIUI/HyperOS) and on other phones with their own autostart
manager, steps 1 and 3 need one extra switch in the phone's settings: without it
the phone refuses to start the song reader (and the boot start), so the status
would stay on the idle text. The app notices it, says so in red on step 1 and
step 3, and its button opens the right screen.

No ADB, no root, no restricted-setting unlock: every step is a normal Settings
screen or a standard system dialog. The APK requests exactly five permissions and
nothing else — you can verify with
`aapt2 dump badging spotify-to-whatsapp-android.apk`.

Full documentation (architecture, permission table, why a WebView, troubleshooting)
is in **[`for android/README.md`](for%20android/README.md)**.

Rebuild it with one command:

```bash
cd "for android" && bash build-apk.sh
```

---

## 6. Testing

Both builds ship a real, runnable test suite — nothing is claimed that has not
been executed.

### Windows

```bash
cd "for pc"
npm test            # 89 unit tests (config, formatting, privacy, encryption, i18n, HUD, tray)
npm run test:secure # encryption smoke test: REAL DPAPI key + vault lock/unlock in a temp folder
npm run test:media  # reads the live Windows media sessions (play something on Spotify first)
npm run test:whatsapp
```

Expected output of the two suites that need no Spotify account:

```text
tests 89 / pass 89 / fail 0
RESULT: all encryption smoke checks passed.
```

`test:secure` is worth reading: it creates a throwaway folder, wraps a real key
with DPAPI, encrypts a fake session and a config, then checks that **no plaintext
phone number or session content is left anywhere on disk**.

### Android

```bash
cd "for android"
bash build-apk.sh tests    # 15 JVM unit tests for the shared logic (formatting, limits, filtering, log sanitising)
```

The tests assert the same behaviour as the Windows formatter (same placeholders,
same 50/139-character limits, same "a playing session wins" rule), so the two
implementations cannot drift apart silently.

---

## 7. How the two builds talk to WhatsApp

Both use **WhatsApp Web as a linked device** — the only supported way to change
your own profile programmatically. The difference is the host:

- **Windows** launches Chromium through `whatsapp-web.js`, and the program injects
  calls to WhatsApp Web's internal modules.
- **Android** hosts WhatsApp's own web client inside an in-app `WebView` and
  injects the *same* internal calls.

The internal calls, in order of preference, are:

1. `WAWebMexUpdateTextStatusJob.mexUpdateTextStatus(text, emoji, duration)` —
   the GraphQL mutation WhatsApp Web itself uses. It sets text, emoji and
   duration in one request, which is what makes the status actually **visible**
   on modern phones (a status without a duration is stored but never shown).
2. `WAWebContactTextStatusBridge.setTextStatus(...)` — same mutation, helper route.
3. `WAWebSetAboutJob.sendSetAbout(...)` — the legacy IQ. It saves the old About
   field and may not appear on modern phones; the app logs when it has to use it.

Reading your previous description uses the matching GraphQL text-status fetch, so
the "restore" puts back exactly what your phone was showing.

---

## 8. Troubleshooting

| | Windows | Android |
|---|---|---|
| **Nothing is detected** | Spotify must be *playing* and visible in the Windows media panel (`npm run test:media`) | Start playback once with the screen on; check **Activity log** in the app |
| **Pairing code refused** | Codes expire in a few minutes: restart the app for a new one | Re-open the linking screen for a fresh code |
| **Stopped updating overnight** | Check the tray UI is still running | Complete step 4 (battery exemption), plus your manufacturer's own "autostart" screen (on MIUI/HyperOS it is required even for the song reader, see §5) |
| **The description stayed on a song** | Start the app again: auto-repair finishes the job | Same: auto-repair runs at the next start |
| **Moving to another machine/phone** | The encrypted number/session belongs to your Windows account: delete `.secrets/` and `.wwebjs_auth/` and pair again | Unlink the device from WhatsApp and pair again |
| **Detailed logs** | `app.log`, `app.err`, or `start.bat` for the live HUD | **Activity log** button in the app |

---

## 9. Disclaimer

Both builds are **unofficial** and use WhatsApp Web the same way a browser does,
to change **your own** profile fields only. They do not read chats, contacts or
messages and do not message anyone. Automating your own account can, in principle,
be against WhatsApp's terms of service: use it responsibly and at your own risk.
