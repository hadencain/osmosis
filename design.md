# OSMOSIS.EXE — Design Document

## Concept

Two videos are loaded simultaneously. Each frame, bright pixels from one video bleed into the corresponding dark regions of the other. The contamination spreads based on luminance gradients — the brighter source wins, slowly. Over time both timelines eat each other until neither is recognizable. At low bleed rates the process takes minutes; at high rates it happens in seconds. The final state is a composite that belongs to neither source.

The metaphor is osmosis: material moves from high concentration (brightness) to low, across a membrane (the canvas), until equilibrium destroys both.

---

## Core Mechanic

Two video elements play simultaneously into two offscreen canvases. Each frame, pixel data is read from both. Per pixel: luminance is computed for each source. The brighter source bleeds a small amount of its color into the contamination buffer. The contamination buffer persists between frames — it is the running output, not a fresh composite. Over time it drifts away from both originals.

To maintain performance, both videos are internally processed at a capped resolution (854×480 max), with the output scaled up to display size.

---

## Algorithm

```js
// --- SETUP ---
const INTERNAL_W = 854, INTERNAL_H = 480; // cap for performance

const videoA = document.createElement('video');
const videoB = document.createElement('video');

const canvasA = new OffscreenCanvas(INTERNAL_W, INTERNAL_H);
const canvasB = new OffscreenCanvas(INTERNAL_W, INTERNAL_H);
const ctxA = canvasA.getContext('2d');
const ctxB = canvasB.getContext('2d');

// contaminationBuffer is the persistent output — initialized to videoA's first frame
let contaminationBuffer; // ImageData, INTERNAL_W × INTERNAL_H
let bufInitialized = false;

// --- PER FRAME ---
ctxA.drawImage(videoA, 0, 0, INTERNAL_W, INTERNAL_H);
ctxB.drawImage(videoB, 0, 0, INTERNAL_W, INTERNAL_H);

const dataA = ctxA.getImageData(0, 0, INTERNAL_W, INTERNAL_H).data;
const dataB = ctxB.getImageData(0, 0, INTERNAL_W, INTERNAL_H).data;

if (!bufInitialized) {
  contaminationBuffer = ctxA.getImageData(0, 0, INTERNAL_W, INTERNAL_H);
  bufInitialized = true;
}

const buf = contaminationBuffer.data;

for (let i = 0; i < INTERNAL_W * INTERNAL_H; i++) {
  const r = i * 4, g = r + 1, b = r + 2;

  // Luminance (ITU-R BT.601)
  const lumA = 0.299 * dataA[r] + 0.587 * dataA[g] + 0.114 * dataA[b];
  const lumB = 0.299 * dataB[r] + 0.587 * dataB[g] + 0.114 * dataB[b];

  let t = 0;
  let srcR, srcG, srcB;

  if (direction === 'A_TO_B' || direction === 'BIDIRECTIONAL') {
    if (lumA > lumB) {
      t = bleedRate * ((lumA - lumB) / 255);
      srcR = dataA[r]; srcG = dataA[g]; srcB = dataA[b];
      buf[r] = buf[r] + (srcR - buf[r]) * t;
      buf[g] = buf[g] + (srcG - buf[g]) * t;
      buf[b] = buf[b] + (srcB - buf[b]) * t;
    }
  }

  if (direction === 'B_TO_A' || direction === 'BIDIRECTIONAL') {
    if (lumB > lumA) {
      t = bleedRate * ((lumB - lumA) / 255);
      srcR = dataB[r]; srcG = dataB[g]; srcB = dataB[b];
      buf[r] = buf[r] + (srcR - buf[r]) * t;
      buf[g] = buf[g] + (srcG - buf[g]) * t;
      buf[b] = buf[b] + (srcB - buf[b]) * t;
    }
  }
}

// Write contamination buffer to display canvas (scaled to display size)
displayCtx.putImageData(contaminationBuffer, 0, 0);
```

**Contamination meter calculation (once per second):**
```js
// Sample N random pixels, compare to original videoA frame
// % significantly altered = pixels where color distance from original > threshold
function computeContaminationPercent(original, current, threshold = 30) {
  let altered = 0;
  const sampleSize = 1000;
  for (let i = 0; i < sampleSize; i++) {
    const idx = Math.floor(Math.random() * (INTERNAL_W * INTERNAL_H)) * 4;
    const dr = Math.abs(current[idx]   - original[idx]);
    const dg = Math.abs(current[idx+1] - original[idx+1]);
    const db = Math.abs(current[idx+2] - original[idx+2]);
    if (dr + dg + db > threshold) altered++;
  }
  return altered / sampleSize;
}
```

---

## Controls

| Parameter | Range | Default | Effect |
|-----------|-------|---------|--------|
| Bleed Rate | 0.001–0.1 | 0.01 | Per-frame contamination intensity. 0.001 = very slow, 0.1 = fast |
| Direction | A→B / B→A / Bidirectional | Bidirectional | Which source can bleed into the other |
| Reset | Button | — | Clears contamination buffer back to current videoA frame |

Internal resolution is fixed at 854×480 — not user-facing.

---

## Browser APIs

- **Canvas 2D**: pixel read/write on two offscreen canvases + display canvas
- **HTMLVideoElement**: two instances playing simultaneously
- **MediaRecorder**: for export
- **FileReader**: for loading both video files as data URLs (same as GLITCH.EXE)

---

## Performance Notes

- Per-pixel loop is O(width × height) = ~409,920 iterations per frame at 854×480
- This is the primary cost. At 30fps that's ~12M pixel operations/sec — acceptable in a typed array loop
- **Never process at full 1080p** — it will drop below 30fps. The 854×480 cap is hard
- If FPS drops below 25: skip every other frame's pixel processing (draw to canvas but don't run the bleed loop)
- Use `Uint8ClampedArray` arithmetic throughout — no object allocation in the hot loop
- `getImageData` → typed array → math → `putImageData` is the correct pattern. Do not use `ctx.globalAlpha` or `drawImage` compositing — they cannot produce the luminance-gated per-pixel bleed behavior

---

## Edge Cases

| Case | Handling |
|------|----------|
| Videos have different dimensions | Scale both to fit the smaller of the two at INTERNAL_W/H bounds |
| Videos have different lengths | When one ends: pause that video, freeze its frame, continue contamination using frozen frame |
| One video not yet loaded | Block playback start until both videos are loaded and ready |
| Reset pressed during playback | Re-initialize `contaminationBuffer` from current videoA frame; do not pause |
| Same video loaded for A and B | Works fine — contamination still occurs near motion edges where luminance fluctuates |
| Large files (>200MB) | FileReader may be slow; show loading progress same as GLITCH.EXE |

---

## UI / Aesthetic

Same terminal aesthetic as GLITCH.EXE.

**Osmosis-specific additions:**
- Two side-by-side drop zones at top: `[DROP VIDEO A]` and `[DROP VIDEO B]` — both must be loaded before playback unlocks
- Single output canvas below the drop zones showing the contamination buffer
- Small source previews: thumbnails of current videoA and videoB frames shown in corners of output or in a strip above it (optional — can be omitted for simplicity)
- **Contamination meter**: a horizontal bar below the output labeled `CONTAMINATION: XX%` — updates once per second via the sampling method above
- Direction selector: three toggle buttons (A→B / BIDIRECTIONAL / B→A)
- Debug panel shows current bleed rate, frame time, and contamination %

---

## Done Criteria

- [ ] Two separate video files loadable via drag/drop or file picker (separate zones)
- [ ] Both videos play simultaneously after both are loaded
- [ ] Pixel bleed visibly occurs over time at default settings
- [ ] At bleed rate 0.1, contamination is dramatic within a few seconds
- [ ] At bleed rate 0.001, contamination is nearly imperceptible per frame
- [ ] Direction control changes which source bleeds
- [ ] Reset correctly re-initializes the buffer from videoA
- [ ] Contamination meter responds to actual pixel data (not a fake counter)
- [ ] Different-length video case handled (shorter video freezes at end)
- [ ] Export produces a valid `.webm` of the contamination buffer output
