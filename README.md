# v0dev Vision Analysis Service

## Purpose

This is the **vision frontend** of the v0dev compiler pipeline.

It transforms:
```
video.mp4 → observations.json
```

Observations are **probabilistic claims** about:
- Layout structure
- Motion patterns
- Temporal sections

These claims feed into downstream services that:
1. Generate QA questions for low-confidence claims
2. Fuse accepted claims into the CreativeSpec (canonical IR)

---

## Architecture

```
┌─────────────┐
│  video.mp4  │
└──────┬──────┘
       ↓
┌──────────────────┐
│ 1. Metadata      │ → fps, duration, resolution
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 2. Frame Extract │ → frame_XXXX.png + timestamps.json
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 3. Motion        │ → optical flow, energy, direction
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 4. Segmentation  │ → section boundaries (SSIM + motion)
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 5. Observations  │ → observations.json
└──────────────────┘
```

---

## Output Contract

**File:** `data/output/observations.json`

**Schema:**
```json
[
  {
    "id": "obs_sec0_fade",
    "sectionId": "sec_0",
    "category": "motion",
    "claim": "Hero title fades in and moves upward",
    "confidence": 0.78,
    "evidence": {
      "frameRange": [12, 30],
      "keyframes": ["frame_0012.png", "frame_0025.png"]
    }
  }
]
```

**Fields:**
- `id`: Unique observation identifier
- `sectionId`: Section this observation belongs to
- `category`: `motion` | `layout` | `temporal`
- `claim`: Human-readable description
- `confidence`: Float in [0, 1]
- `evidence`: Frame ranges and keyframe references

---

## Setup

```bash
# Install dependencies
pip install -r requirements.txt

# Verify ffmpeg is installed
ffmpeg -version
```

---

## Usage

```bash
# Place video in data/input/
cp your_video.mp4 data/input/sample.mp4

# Run pipeline
python scripts/run_pipeline.py

# Output appears in:
# - data/frames/          (extracted frames)
# - data/output/          (observations.json)
```

---

## Pipeline Stages

### 1. Metadata Extraction
**Module:** `src/ingest/metadata.py`

Uses ffprobe to extract:
- Duration (seconds)
- FPS
- Resolution (width, height)
- Total frame count

### 2. Frame Extraction
**Module:** `src/frames/extract_frames.py`

- Samples frames at fixed FPS (default: 5fps)
- Saves as `frame_0000.png`, `frame_0001.png`, ...
- Generates `timestamps.json` mapping frame index → timestamp

### 3. Motion Analysis
**Module:** `src/motion/optical_flow.py`

Using OpenCV Farnebäck dense optical flow:
- Computes per-pixel flow vectors (u, v)
- Computes motion magnitude per frame
- Extracts global motion energy
- Detects median direction (Δx, Δy)

### 4. Section Segmentation
**Module:** `src/sections/segmenter.py`

Detects scene boundaries using:
- SSIM (Structural Similarity) between frames
- Motion energy discontinuities

Boundary score: `B_t = α * E_t + β * (1 - SSIM)`

### 5. Observation Builder
**Module:** `src/observations/builder.py`

Generates claims based on:
- Fade-in detection (brightness delta)
- Slide-up detection (vertical optical flow)
- Scroll detection (sustained vertical motion)

Each claim includes confidence score based on signal strength.

---

## Configuration

Edit `config/defaults.yaml`:
```yaml
sampling:
  fps: 5  # Frame sampling rate

motion:
  flow_algorithm: farneback
  energy_threshold: 50

segmentation:
  ssim_threshold: 0.85
  alpha: 0.6  # Motion weight
  beta: 0.4   # SSIM weight

observations:
  min_confidence: 0.5
```

---

## Design Principles

1. **Deterministic**: Same video → same observations
2. **No randomness** (unless explicitly seeded)
3. **Evidence-based**: Every claim has frame references
4. **Explainable**: Heuristics over black-box models
5. **Probabilistic**: Confidence scores, not binary truth

---

## Tech Stack

- **Python 3.9+**
- **OpenCV**: Frame extraction, optical flow
- **scikit-image**: SSIM computation
- **ffmpeg-python**: Video metadata
- **NumPy**: Numerical processing

---

## Future Extensions

- YOLO object detection (for button/text detection)
- CLIP embeddings (for semantic layout)
- Deep flow models (for complex motion)

These will be added after the heuristic baseline is validated.

---

## License

See LICENSE file.