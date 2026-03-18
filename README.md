# VidGuard — Deepfake Detector

**Real-time AI deepfake detection for TikTok, YouTube Shorts, and Instagram Reels.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat&logo=python&logoColor=white)](https://python.org)
[![Flask](https://img.shields.io/badge/Flask-3.x-000000?style=flat&logo=flask&logoColor=white)](https://flask.palletsprojects.com)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.x-EE4C2C?style=flat&logo=pytorch&logoColor=white)](https://pytorch.org)
[![Chrome Extension](https://img.shields.io/badge/Chrome-Manifest%20V3-4285F4?style=flat&logo=googlechrome&logoColor=white)](https://developer.chrome.com/docs/extensions/mv3/)

[Visit Our Web App]
https://vidguard-435925497959.us-central1.run.app

</div>

---

## Overview

VidGuard is a full-stack deepfake detection system. It can analyse both full videos and individual frames, and ships with a Chrome extension that lets you scan short-form videos directly from your browser toolbar while browsing TikTok, YouTube Shorts, or Instagram Reels.

## Repository structure

```
Main/
├── vidguard-extension/
│   ├── manifest.json     # Chrome MV3 manifest
│   ├── background.js     # Service worker — API calls
│   ├── content.js        # Frame capture on demand
│   ├── popup.html        # Extension popup UI
│   ├── popup.js          # Popup logic & state machine
│   ├── overlay.css       # Styles
│   └── icons/            # Extension icons (16 / 48 / 128 px)
│
└── README.md
```
---

## Chrome Extension

### Installation (developer mode)

1. Download [`vidguard-extension.zip`](vidguard-extension.zip)
2. Unzip it
3. Open Chrome and go to `chrome://extensions`
4. Enable **Developer mode** (toggle, top right)
5. Click **Load unpacked** and select the `vidguard-extension/` folder
6. The VidGuard shield icon appears in your toolbar

### How to use

1. Navigate to **TikTok**, **YouTube Shorts**, or **Instagram Reels**
2. Click the VidGuard shield icon in the Chrome toolbar
3. Click **Scan current video**
4. The result appears in the popup:
   - 🔴 **DEEPFAKE** — manipulated content detected
   - 🟢 **AUTHENTIC** — no manipulation detected
5. Click **Rescan** to scan again at a different frame

### Extension architecture

| File | Role |
|---|---|
| `manifest.json` | MV3 manifest — permissions: `activeTab`, `storage`, `scripting` |
| `background.js` | Service worker — converts canvas JPEG to FormData, calls `/api/predict-image`, parses response |
| `content.js` | Injected into TikTok/YouTube/Instagram — captures the largest visible video frame via Canvas on demand |
| `popup.html` | Extension popup — 4 states: idle → loading → result → error |
| `popup.js` | State machine — orchestrates tab messaging, background messaging, and UI updates |

### Supported platforms

| Platform | URL match | Detection |
|---|---|---|
| TikTok | `tiktok.com/*` | All videos |
| YouTube | `youtube.com/shorts/*` | Shorts only |
| Instagram | `instagram.com/reels/*` | Reels only |

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

<div align="center">
Built with the <a href="https://github.com/ondyari/FaceForensics">FaceForensics++</a> dataset and the <a href="https://arxiv.org/abs/1610.02357">Xception</a> architecture.
</div>
