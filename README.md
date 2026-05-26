# OSMOSIS.EXE

Single-file browser tool for pixel contamination between two videos.

Drop two video files in. Pixels from one bleed into the other based on luminance — bright areas overwrite the contamination buffer at a rate you control. Over time the two videos slowly merge.

## Usage

Open `osmosis.html` in a browser. No install, no server, no dependencies.

1. Drop **Video A** and **Video B** into the two zones (or click to browse)
2. Hit **PLAY**
3. Watch the contamination meter — it tracks how much the buffer has diverged from Video A's initial frame
4. Hit **RESET** to restore the buffer to Video A's current frame and start again

## Controls

| Control | What it does |
|---------|-------------|
| `A→B` / `BIDI` / `B→A` | Direction of bleed — which video's bright pixels contaminate the buffer |
| `BLEED` slider | Rate of contamination per frame (0.001–0.1) — low is glacial, high is fast |
| `LOOP` | Toggle whether videos loop when they end |
| `RESET` | Snapshot Video A's current frame into the contamination buffer |
| `REC` / `STOP REC` | Record the canvas output to `osmosis.webm` |

## How it works

Each frame, both videos are drawn to offscreen canvases. Per-pixel luminance is compared between them. If a pixel in the source video is brighter than the same pixel in the contamination buffer, it bleeds into the buffer at a rate proportional to the luminance difference × bleed rate. The display always shows the contamination buffer — not the raw video.

At low bleed rates the effect is slow and spectral. At high rates it's aggressive and immediate.

## Export

Hit **REC** while playing to capture the output. Stops and downloads `osmosis.webm` on **STOP REC** or when playback is stopped. Uses `MediaRecorder` with VP9 where supported.

## Browser support

Requires OffscreenCanvas and MediaRecorder — Chrome/Edge work well. Firefox has partial support. No Safari.
