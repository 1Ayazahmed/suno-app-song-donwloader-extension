<div align="center">

<img src="icons/icon128.png" width="96" alt="Suno Advanced Downloader logo" />

# Suno Advanced Downloader

**A Chrome extension (Manifest V3) that downloads the track playing on [suno.com](https://suno.com) as MP3 — with a side panel for live progress, a download queue, history and settings.**

![Manifest V3](https://img.shields.io/badge/Manifest-V3-e8b04b)
![Chrome 116+](https://img.shields.io/badge/Chrome-116%2B-4285F4)
![Version](https://img.shields.io/badge/version-2.3.0-2ea44f)
![No build step to install](https://img.shields.io/badge/install-load%20unpacked-lightgrey)

</div>

---

## Overview

Suno Advanced Downloader adds a **side panel** and a **one-click floating button** to Suno. It captures the playing track, optionally converts it to MP3 with a bundled LAME encoder, and saves it to a folder in your Downloads directory — while showing every pipeline stage (detect → record → encode → save) in real time.

No accounts, no external servers, no analytics. Everything runs locally inside the extension.

## Features

| | |
|---|---|
| **One-click download** | A floating button above the player grabs the current track without opening any UI |
| **Side panel** | Live now-playing card, per-stage progress bar, and an activity log |
| **MP3 output** | Real MP3 encoding via `lamejs`, with a selectable bitrate (128 / 192 / 320 kbps) |
| **Graceful fallback** | If MP3 encoding fails, the audio is saved as WebM rather than losing the download |
| **Smart detection** | Reads title, artist and cover art from the player, page metadata, and the page URL |
| **Auto-play** | Pressing **Start Download** begins playback itself if the song is paused |
| **Queue** | Add tracks to a queue; it processes them one at a time with retry support |
| **History** | Keeps the last *N* downloads with file size, format and timestamp, plus a **Play** shortcut |
| **Settings** | Output folder, bitrate, auto-MP3 conversion, notifications, history limits |
| **Notifications & badge** | Desktop notification on completion; toolbar badge shows queue activity |

## How it works

```
┌──────────────┐   ┌────────────────┐   ┌───────────────────┐   ┌──────────────┐
│ Content      │   │ Background     │   │ Offscreen         │   │ chrome.      │
│ script       │──▶│ service worker │──▶│ document          │──▶│ downloads    │
│ (suno.com)   │   │ (queue/state)  │   │ (lamejs encoder)  │   │ (disk)       │
└──────────────┘   └────────────────┘   └───────────────────┘   └──────────────┘
```

1. The **content script** finds Suno's `<audio>` element, rewinds it to `0:00` and records it using `captureStream()` + `MediaRecorder`.
2. The recording is sent to the **background service worker** as base64 (`chrome.runtime.sendMessage` cannot carry `ArrayBuffer`s).
3. The **offscreen document** decodes the audio and encodes MP3 at the configured bitrate using the bundled `lamejs`.
4. `chrome.downloads` saves the file to `Downloads/<folder>/<song>.mp3`, and the history entry gets a **Play** button that opens it in your default player.

Because capture happens through live playback, **a 3-minute song takes roughly 3 minutes**, and the Suno tab must stay open and unmuted during that time.

## Installation

### From source (recommended)

```bash
git clone https://github.com/1Ayazahmed/suno-app-song-donwloader-extension.git
cd suno-app-song-donwloader-extension
npm install
npm run build
```

Then:

1. Open `chrome://extensions`
2. Enable **Developer mode** (top right)
3. Click **Load unpacked** and select the cloned folder
4. Pin the extension, open a song on `suno.com`, and click the toolbar icon to open the side panel

> The extension is not published to the Chrome Web Store. Chrome 116 or newer is required (side panel + offscreen document APIs).

## Usage

**One-click:** hover the player on any Suno song and click the gold download button. The icon spins while downloading, then turns green.

**Side panel:** click the toolbar icon →

1. The now-playing card shows the detected track
2. Press **Start Download (automatic)** — playback starts on its own if needed
3. Watch each stage complete and follow the activity log
4. **Play** appears under the saved file when the download finishes

Open song pages in other tabs and use **Add to queue** to batch them.

## Settings

| Setting | Default | Description |
|---|---|---|
| Auto-record | on | Start recording as soon as a download is requested |
| Convert to MP3 | on | Encode with `lamejs`; disable to save WebM only |
| Quality | 192 kbps | MP3 bitrate: 128 / 192 / 320 |
| Download folder | `Suno Downloads` | Sub-folder inside your Downloads directory |
| Notifications | on | Desktop notification on completion |
| Keep history | on | Store finished downloads |
| Max history items | 100 | Older entries are pruned automatically |

## Project structure

```
manifest.json            Manifest V3 definition, permissions and entry points
icons/                   PNG icons + make-icons.js generator
src/
├── background/index.js  Service worker: queue, pipeline orchestration, storage, badge
├── content/
│   ├── index.js         Page script: detection, recording, floating button + panel
│   └── bundle.js        Generated esbuild bundle loaded by the manifest
├── offscreen/           Offscreen document: audio fetch, MP3 encoding, blob URLs
├── popup/               Options page
├── sidepanel/           Side panel UI (queue, history, settings, activity log)
└── shared/
    ├── constants.js     Message types, storage keys, defaults, shared formatters
    └── storage.js       chrome.storage helpers
```

## Development

```bash
npm run build        # bundle src/content/index.js → src/content/bundle.js
npm run build:watch  # rebuild on change
npm run check        # build + syntax-check every entry point
```

`src/content/bundle.js` is a **generated artifact** and is committed because the manifest loads it directly. Rebuild after editing the content script or anything it imports from `src/shared/`.

After changing the content script, click **Reload** on the extension card in `chrome://extensions`, then refresh the Suno tab.

### Permissions

| Permission | Why it is needed |
|---|---|
| `storage` | Persist queue, history and settings |
| `downloads`, `downloads.open` | Save files and open them in the default player |
| `offscreen` | Run the MP3 encoder and in-memory blob URLs outside the service worker |
| `sidePanel` | Provide the side panel UI |
| `notifications` | Report completion or errors |
| `host_permissions: suno.com` | Inject the content script and read the player |

## Known limitations

- Recording is **real-time and lossy** — the output is a re-encode of playback, not a copy of the original master file.
- The Suno tab must remain open, playing and unmuted for the duration of the recording.
- "Add to queue" records whatever is *playing* when that item's turn comes; it does not navigate to the queued song.
- Detection relies on Suno's current DOM. A major site redesign can require selector updates in `src/shared/constants.js`.
- Chrome only — no Firefox or Safari support.

## Disclaimer

This project is intended for **personal use** — for example, archiving tracks you created or have the rights to. It is not affiliated with, endorsed by, or supported by Suno. You are responsible for complying with Suno's Terms of Service and applicable copyright law in your jurisdiction.

## Contributing

Issues and pull requests are welcome. Please run `npm run check` before submitting, keep changes scoped, and describe the Suno UI context when reporting a detection bug.

---

<div align="center">
<sub>Built for creators who want their own tracks back on their own disk.</sub>
</div>
