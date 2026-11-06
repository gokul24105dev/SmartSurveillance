# SmartFace – Criminal Record Face Match + Weapon Detection

A single-file, browser-based prototype that checks a live camera feed against a criminal-record database (face recognition) and scans the same feed for weapons (YOLO object detection). Everything runs locally in the browser: no backend, and no images leave the machine.

> **Status:** prototype / proof of concept. Not validated for real-world law-enforcement or security decisions. See [Limitations](#limitations-and-responsible-use).

---

## Features

| Feature | Description |
|---|---|
| **Face match** | Detects a face from the webcam and compares it with enrolled records. Shows **MATCH FOUND** (name, record ID, distance, similarity) or **NOT FOUND**. |
| **Record import** | Load records from JSON (`[{id, name, img}]`) or bulk-import from CSV (`name,image`). |
| **Live enroll** | Capture a face from the camera and add it under a name; add more captures per person for better accuracy. |
| **Weapon detection** | Runs a YOLOv8 / YOLO11 `.onnx` model on the live feed, draws boxes and labels, and shows a **WEAPON DETECTED** banner. |
| **Strictness slider** | Tunes the face-match distance threshold (0.40 – 0.70, default 0.55). |
| **Export / persistence** | Records are stored in browser `localStorage` and can be exported as JSON. |
| **Alert control room** | *Placeholder button – not functional yet.* |
| **Web scrape person details** | *Placeholder button – not functional yet.* |

---

## Tech stack

- **Face detection / recognition:** [`@vladmandic/face-api`](https://github.com/vladmandic/face-api) (TinyFaceDetector, 68-point landmarks, 128-D face descriptors)
- **Object detection:** [`onnxruntime-web`](https://onnxruntime.ai/) (WASM, CPU) running a YOLO ONNX model
- Plain HTML / CSS / JavaScript – one file: `SmartFace_CriminalRecord.html`

---

## Getting started

### Requirements
- A modern browser (Chrome, Edge, Firefox, Safari)
- A webcam
- Python 3 (only to serve the file locally)
- Internet access the **first** time (face models and ONNX runtime load from the jsDelivr CDN)

### Run

Browsers only allow camera access on `https` or `localhost`, so serve the file locally:

```bash
cd <folder containing SmartFace_CriminalRecord.html>
python -m http.server 8000
```

Open: <http://localhost:8000/SmartFace_CriminalRecord.html>

Wait for the **models: ready** pill, then click **Start camera** and allow camera permission.

---

## Usage

### 1. Load criminal records

**Option A – JSON** (right panel → *Upload record JSON*)

```json
[
  { "id": "CR-1001", "name": "Person One", "img": "https://example.com/photo1.jpg" },
  { "id": "CR-1002", "name": "Person Two", "img": "person2.jpg" }
]
```

- `img` can be a `data:image/...` URI, an `http(s)` URL, or a filename.
- If any entries use filenames, click **2 · Choose photo files** and select those images.
- Photos with no detectable face are skipped and reported.
- A previously exported record file (with face descriptors) can also be re-imported.

**Option B – CSV** (*Bulk import from CSV*)

```csv
name,image
Person One,person1.jpg
Person One,person1_side.jpg
Person Two,person2.jpg
```

Repeat a name to add multiple photos of the same person. Use **Sample CSV** to download a template.

**Option C – Enroll from camera:** type a name and click **Capture & enroll**.

### 2. Run a face check
Click **Start live check**. A green box and **MATCH FOUND** banner appear for people in the records; a red box and **NOT FOUND** for everyone else. Adjust **Strictness** if you see false matches (lower the value) or missed matches (raise it).

### 3. Run weapon detection

1. In the **Weapon detection** card, click **Load YOLO model (.onnx)**.
2. Enter the **class names** (comma-separated, in the same order the model was trained), e.g. `gun, knife`.
3. Choose the **input size** to match how the model was exported (default 640).
4. Start the camera, then click **Start weapon detection**.
5. Tune the **Confidence** slider (default 0.45).

Face check and weapon detection can run at the same time or independently.

#### Exporting your own model

```bash
pip install ultralytics
yolo export model=best.pt format=onnx imgsz=640
```

- Supported: **YOLOv8 / YOLO11** ONNX exports (output shape `[1, 4+classes, N]`).
- Not supported: YOLOv5-style output (separate objectness score).

#### Trying it with the sample model
`yolov8n.onnx` (COCO, 80 classes) works for testing the pipeline only. COCO has **no gun class**, so it will flag everyday objects (people, cups, phones) as "weapons". Train a custom model on a weapons dataset (e.g. from Roboflow Universe) for real use.

---

## How it works

```
Camera frame ──┬─► face-api: detect → landmarks → 128-D descriptor
               │        └─► Euclidean distance to every enrolled sample
               │              └─► ≤ threshold → MATCH, else NOT FOUND
               │
               └─► YOLO ONNX: letterbox → CHW float tensor → inference
                        └─► confidence filter → per-class NMS → boxes + banner
```

Both run in a single `requestAnimationFrame` loop with a busy-guard so frames never overlap.

---

## Project structure

```
SmartFace_CriminalRecord.html   # entire app (HTML + CSS + JS)
README.md
yolov8n.onnx                    # optional sample model (COCO)
coco_classes.txt                # optional class names for the sample model
```

---

## Limitations and responsible use

- **Accuracy:** face matching can produce false positives and false negatives, which vary with lighting, angle, image quality, and across demographic groups. A match is a *lead to verify by a human*, never proof of identity.
- **Weapon detection** quality depends entirely on your trained model and data; expect false alarms (phones, tools, umbrellas) and misses.
- **Performance:** inference runs on CPU in the browser, so expect a few frames per second. The model must be re-loaded after each page refresh.
- **Privacy / legal:** biometric and criminal-record data is sensitive. Use only with proper authorisation and in line with local law (e.g. India's DPDP Act, GDPR where applicable). Records are stored unencrypted in browser `localStorage`.
- **Placeholders:** the *Alert control room* and *Web scrape person details* buttons have no behaviour yet. If you implement scraping, restrict it to authorised, official or public-record sources.

---

## Roadmap ideas

- Wire **Alert control room** to a backend (WebSocket / REST / SMS) with name, record ID, snapshot, camera ID and timestamp.
- Auto-trigger alerts on a confirmed weapon detection or high-confidence match.
- Multi-frame confirmation to reduce banner flicker.
- Persist the ONNX model in IndexedDB.
- WebGL/WebGPU execution provider for faster inference.
- Server-side storage with encryption and audit logs.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Camera won't start | Serve from `http://localhost` or `https`; allow camera permission in the browser. |
| `models: failed` | Check internet access (CDN) and reload. |
| Imported photos skipped | No face found in the image; use a clear, front-facing photo. |
| Weapon model fails to load | Re-export with `format=onnx` from YOLOv8/YOLO11; avoid YOLOv5 exports. |
| Wrong labels on boxes | Class names must match the model's training order. |
| Nothing detected | Lower the confidence slider and check the input size matches the export. |
| Slow detection | Use a smaller model (`yolov8n`) or a smaller input size (416 / 320). |
