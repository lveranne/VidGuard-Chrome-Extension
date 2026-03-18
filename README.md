# VidGuard — Deepfake Detector

**Real-time AI deepfake detection for TikTok, YouTube Shorts, and Instagram Reels.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat&logo=python&logoColor=white)](https://python.org)
[![Flask](https://img.shields.io/badge/Flask-3.x-000000?style=flat&logo=flask&logoColor=white)](https://flask.palletsprojects.com)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.x-EE4C2C?style=flat&logo=pytorch&logoColor=white)](https://pytorch.org)
[![Chrome Extension](https://img.shields.io/badge/Chrome-Manifest%20V3-4285F4?style=flat&logo=googlechrome&logoColor=white)](https://developer.chrome.com/docs/extensions/mv3/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](LICENSE)

[Web App](https://vidguard-435925497959.us-central1.run.app) · [Install Extension](#-chrome-extension) · [API Docs](#-api-reference)

</div>

---

## Overview

VidGuard is a full-stack deepfake detection system built on the **Xception** model trained on the FaceForensics++ dataset. It exposes a Flask REST API that can analyse both full videos and individual frames, and ships with a Chrome extension that lets you scan short-form videos directly from your browser toolbar while browsing TikTok, YouTube Shorts, or Instagram Reels.

```
User clicks Scan in the Chrome popup
        │
        ▼
content.js captures the current video frame via Canvas
        │
        ▼
background.js POSTs the JPEG to Flask /api/predict-image
        │
        ▼
helper.py  →  OpenCV face detection  →  Xception inference
        │
        ▼
Verdict (REAL / FAKE) + confidence % shown in the popup
```

---

## Repository structure

```
MMDDS/
├── backend/
│   ├── app.py            # Flask API — all endpoints
│   ├── helper.py         # Xception model, face detection, inference loop
│   └── models/
│       └── full_raw.p    # FaceForensics++ Xception checkpoint (not committed)
│
├── vidguard-extension/
│   ├── manifest.json     # Chrome MV3 manifest
│   ├── background.js     # Service worker — API calls
│   ├── content.js        # Frame capture on demand
│   ├── popup.html        # Extension popup UI
│   ├── popup.js          # Popup logic & state machine
│   ├── overlay.css       # Styles
│   └── icons/            # Extension icons (16 / 48 / 128 px)
│
├── frontend/             # React web app (Vite + Tailwind)
│   └── src/
│       ├── pages/DashboardPage.tsx
│       └── components/
│           ├── HeroSection.tsx
│           └── DemoModal.tsx
│
└── README.md
```

---

## Backend

### Requirements

| Package | Version |
|---|---|
| Python | ≥ 3.10 |
| Flask | ≥ 3.0 |
| flask-cors | ≥ 4.0 |
| PyTorch | ≥ 2.0 |
| torchvision | ≥ 0.15 |
| opencv-python | ≥ 4.8 |
| Pillow | ≥ 10.0 |
| numpy | **< 2.0** ⚠️ |
| tqdm | any |

> **NumPy < 2.0 is required.** PyTorch's internal NumPy bridge was broken by the NumPy 2.0 release. Pin it with:
> ```bash
> pip install "numpy<2.0" --upgrade
> ```

### Installation

```bash
# Clone the repo
git clone https://github.com/yourname/vidguard.git
cd vidguard/backend

# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# Install dependencies
pip install flask flask-cors torch torchvision opencv-python Pillow tqdm
pip install "numpy<2.0" --upgrade  # critical — do this last

# Place the model checkpoint
mkdir -p models
# Copy full_raw.p into backend/models/
```

### Running the server

```bash
cd backend
python app.py
```

The API starts on `http://127.0.0.1:8080`.

Check it's alive:
```bash
curl http://127.0.0.1:8080/api/health
# {"status":"ok","cuda":false,"model_path":"./models/full_raw.p","numpy":"1.26.4"}
```

---

## API Reference

### `POST /api/predict`
Analyse a **full video file**. Used by the web app dashboard.

**Request** — `multipart/form-data`
| Field | Type | Description |
|---|---|---|
| `video` | File | MP4 / AVI / MOV video |
| `end_frame` | int (optional) | Number of frames to analyse (default 20, max 300) |

**Response**
```json
{
  "label": "FAKE",
  "confidence": 85.3,
  "fake_ratio": 0.7241,
  "avg_fake_conf": 87.1,
  "avg_real_conf": 12.9,
  "frames_with_face": 58,
  "frame_confidences": [
    { "frame": 1, "fake_prob": 91.2, "real_prob": 8.8 }
  ],
  "video_id": "uuid-here",
  "video_url": "/api/video/uuid-here",
  "web_app_url": "https://vidguard-435925497959.us-central1.run.app"
}
```

---

### `POST /api/predict-image`
Analyse a **single JPEG frame**. Used by the Chrome extension.

**Request** — `multipart/form-data`
| Field | Type | Description |
|---|---|---|
| `image` | File | JPEG / PNG image |
| `end_frame` | int (optional) | Frames for synthetic video wrapper (default 10) |

**Response** — same schema as `/api/predict`, `video_url` is always `null`.

---

### `GET /api/video/<video_id>`
Stream the annotated output video (bounding boxes + per-frame labels).

---

### `GET /api/health`
Liveness probe. Returns model path, CUDA status, and NumPy version.

---

## Chrome Extension

### Installation (developer mode)

1. Download [`vidguard-extension.zip`](vidguard-extension.zip)
2. Unzip it
3. Open Chrome and go to `chrome://extensions`
4. Enable **Developer mode** (toggle, top right)
5. Click **Load unpacked** and select the `vidguard-extension/` folder
6. The VidGuard shield icon appears in your toolbar

> The backend must be running locally on port 8080 before scanning.

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

## Web App

The React frontend (`/frontend`) provides a full dashboard for uploading and analysing videos with:
- Drag-and-drop video upload
- Configurable frame count (5–300)
- Per-frame confidence chart
- Suspect frame crops
- Annotated video download

Run locally:
```bash
cd frontend
npm install
npm run dev
```

---

## Model

VidGuard uses the **Xception** architecture fine-tuned on the [FaceForensics++](https://github.com/ondyari/FaceForensics) dataset (`full_raw` compression level).

| Property | Value |
|---|---|
| Architecture | Xception (pretrained on ImageNet, fine-tuned on FF++) |
| Input size | 299 × 299 RGB |
| Output | 2-class softmax (REAL / FAKE) |
| Face detector | OpenCV Haar Cascade (`haarcascade_frontalface_default.xml`) |
| Checkpoint | `full_raw.p` (~100 MB) |

The checkpoint is not included in this repository due to file size. Download it from [Hugging Face](https://huggingface.co/Kklv/xception/resolve/main/full_raw.p) and place it at `backend/models/full_raw.p`.

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `RuntimeError: Numpy is not available` | NumPy ≥ 2.0 breaks PyTorch's bridge | `pip install "numpy<2.0" --upgrade` |
| `AssertionError: start_frame exceeds video length` | Sending a non-video file to the model | Fixed in `helper.py` — graceful return instead |
| `No face detected` | Frame has no visible face | Pause video on a frame with a clear face, then rescan |
| `Cannot reach VidGuard server` | Flask not running | Start backend with `python app.py` |
| Extension not injecting | Wrong URL | Must be on `tiktok.com`, `youtube.com/shorts`, or `instagram.com/reels` |

---

## CORS configuration

The Flask backend accepts all origins by default (`CORS(app, resources={r"/api/*": {"origins": "*"}})`). For production, restrict this to your specific origins:

```python
CORS(app, resources={r"/api/*": {"origins": [
    "https://your-frontend-domain.com",
    "chrome-extension://your-extension-id"
]}})
```

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

<div align="center">
Built with the <a href="https://github.com/ondyari/FaceForensics">FaceForensics++</a> dataset and the <a href="https://arxiv.org/abs/1610.02357">Xception</a> architecture.
</div>
