---
title: OSMOSIS.EXE Design Spec
date: 2026-05-26
status: approved
---

# OSMOSIS.EXE

## Concept

Two videos loaded simultaneously. Each frame, bright pixels from one bleed into the dark regions of the other, based on luminance gradients. The contamination buffer persists between frames — it drifts away from both originals over time. At low bleed rates the process takes minutes; at high rates, seconds. The final state belongs to neither source.

---

## File Structure

Single self-contained `osmosis.html`. No build step, no external dependencies except Courier Prime (Google Fonts). All logic inline.

---

## Architecture

### Video Pipeline

- Two `HTMLVideoElement` instances playing simultaneously
- Each drawn into an `OffscreenCanvas` at 854×480 (hard cap — never process at full resolution)
- Contamination buffer: a persistent `ImageData` (854×480), initialized from videoA's first frame
- Display canvas: sized to fill the layout area, CSS-scaled from the 854×480 buffer

### Render Loop

`requestAnimationFrame` drives everything. Each frame:

1. Draw videoA → offscreen canvasA
2. Draw videoB → offscreen canvasB
3. Read pixel data from both via `getImageData`
4. Run bleed loop on contamination buffer (unless frame-skip active)
5. `putImageData` contamination buffer → display canvas

FPS tracked via timestamp delta. If FPS drops below 25, bleed loop runs every other frame (canvas draw still happens every frame).

### Bleed Algorithm

Per pixel, compute ITU-R BT.601 luminance for both sources. If direction allows, the brighter source bleeds into the contamination buffer by `t = bleedRate * ((lumBrighter - lumDarker) / 255)`. Each channel lerps toward the source by `t`. `Uint8ClampedArray` arithmetic throughout — no object allocation in the hot loop.

```
lumA = 0.299*R + 0.587*G + 0.114*B
lumB = 0.299*R + 0.587*G + 0.114*B

if direction allows A→B and lumA > lumB:
  t = bleedRate * ((lumA - lumB) / 255)
  buf[i] = buf[i] + (srcA[i] - buf[i]) * t

if direction allows B→A and lumB > lumA:
  t = bleedRate * ((lumB - lumA) / 255)
  buf[i] = buf[i] + (srcB[i] - buf[i]) * t
```

### Contamination Meter

Once per second: sample 1000 random pixels, compare each to the original videoA snapshot via Manhattan distance on RGB channels. Pixels where `|dR| + |dG| + |dB| > 30` count as altered. `altered / 1000` = contamination percent. Displayed as a horizontal bar labeled `CONTAMINATION: XX%`.

---

## Controls

| Parameter | Range | Default | Notes |
|-----------|-------|---------|-------|
| Bleed Rate | 0.001–0.1 | 0.01 | Slider. 0.001 = imperceptible per frame, 0.1 = dramatic in seconds |
| Direction | A→B / BIDIRECTIONAL / B→A | BIDIRECTIONAL | Three toggle buttons |
| Reset | Button | — | Re-initializes buffer from current videoA frame; does not pause playback |
| REC / STOP | Button | — | MediaRecorder on display canvas stream → `.webm` download on stop |

Internal resolution is fixed — not user-facing.

---

## Load Gate

Playback and REC locked until both videos fire `canplay`. Drop zones show `[WAITING]` until loaded, `[READY]` after. Play button unlocks only when both are ready.

---

## Edge Cases

| Case | Handling |
|------|----------|
| Different dimensions | Scale both to fit within 854×480 bounds (letterbox/pillarbox if needed) |
| Different lengths | Shorter video pauses at last frame; contamination continues using frozen frame |
| One video not loaded | Playback blocked until both ready |
| Reset during playback | Re-init buffer from current videoA frame; no pause |
| Same video for A and B | Works — contamination occurs near motion edges where luminance fluctuates |
| Large files (>200MB) | Show loading progress on drop zone |

---

## UI / Aesthetic

Terminal aesthetic — monochrome. No color.

- Background: `#0a0a0a`
- Text, borders, accents: `#ffffff`
- Font: Courier Prime, monospace
- Scanline overlay (CSS repeating-linear-gradient, low opacity)
- No border-radius anywhere
- Box shadows: white glow at low opacity

### Layout

```
┌─────────────────────────────────────────────────────┐
│  OSMOSIS.EXE                                        │
├──────────────────────┬──────────────────────────────┤
│  [DROP VIDEO A]      │  [DROP VIDEO B]              │
├──────────────────────┴──────────────────────────────┤
│                                                     │
│              OUTPUT CANVAS                          │
│                                                     │
├─────────────────────────────────────────────────────┤
│  CONTAMINATION: ████████░░░░░░░░░░  42%            │
├─────────────────────────────────────────────────────┤
│  BLEED: [slider]   [A→B] [BIDI] [B→A]  [RESET]    │
│  [REC] / [STOP]                                     │
├─────────────────────────────────────────────────────┤
│  DEBUG: bleed=0.01  fps=30  contamination=42%       │
└─────────────────────────────────────────────────────┘
```

---

## Export

REC button starts `MediaRecorder` capturing the display canvas stream (`canvas.captureStream(30)`). STOP ends recording, collects blobs, and triggers a `.webm` download. REC is disabled until both videos are loaded and playing.

---

## Assembly Order

Build in this sequence — validate each step before moving to the next:

1. Video load + dual playback (file picker/drop, both-ready gate)
2. Offscreen canvases + contamination buffer init
3. Bleed loop + `requestAnimationFrame` render
4. FPS monitor + frame-skip fallback
5. Contamination meter (sampling, 1s interval)
6. Direction control
7. Reset
8. Terminal UI shell (drop zones, canvas, controls, debug panel)
9. REC/STOP export (MediaRecorder → `.webm`)

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
