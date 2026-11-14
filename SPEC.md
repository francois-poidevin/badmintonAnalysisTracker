# refApp1 analysis: findings and plan for a client-only clone

Source: static client JS from refApp1 (build v=647), saved in `src/`.
Only shipped client code was read. No model or data files fetched.

## Original pipeline (all in-browser)
| File | Role |
|---|---|
| `src/analisis.js` | UI view `#/analisis`: upload, progress, results. Loads scripts below. |
| `analisis/baca.js` | Video read: `mp4box` demux + WebCodecs `VideoDecoder`; fps/size/codec; audio onset detection (shuttle hit sound, `bacaOnset`). |
| `analisis/mesin.js` | ML engine: `onnxruntime-web` 1.22, WebGPU (WASM fallback). Two ONNX sessions: `o1` (detector) + `pose`. |
| `analisis/engine.js`, `core.js` | Tracking players/shuttle, court calibration (homography to court dims), rally/score logic. |
| `analisis/pukulan.js` | Stroke (shot) classification from pose + shuttle + audio onset. |
| `analisis/hud.js` | Overlay drawing on canvas + export with `mp4-muxer` 5.2.1 (WebCodecs encode -> MP4). |

## Blocker: models
Models are `.vsm` files: AES-GCM encrypted (magic `VSM2`), key fetched from their
backend, host-locked (domain check; pings Supabase RPC when run elsewhere).
They are a protected asset. Do NOT decrypt/redistribute. Use open models instead.

## Plan for own client-only site (static hosting)
1. Upload: `<input type=file>`; store in OPFS (`navigator.storage.getDirectory()`).
2. Decode: `mp4box` + `VideoDecoder` (or `<video>` + `requestVideoFrameCallback`).
3. Models (open): pose = MoveNet / RTMPose / YOLOv8-pose ONNX; shuttle = custom
   TrackNet-style ONNX (open weights exist) ; run via onnxruntime-web (webgpu, wasm fallback).
4. Court: user clicks 4 corners -> homography -> court coordinates.
5. Logic: rally segmentation (shuttle visibility + audio onsets), stroke
   classification (rules on wrist speed / pose first, ML later), score = manual confirm.
6. AR export: draw frame + overlay on OffscreenCanvas -> `VideoEncoder` -> `mp4-muxer` -> Blob download.
7. Constraints: WebGPU needs Chrome/Edge; process in chunks; COOP/COEP headers for WASM threads.

## Next
- Read `src/analisis/hud.js` (non-minified parts) for overlay/export design.
- Read `src/analisis/core.js` for homography + court constants.
- Scaffold `app/` (Vite or plain ES modules).

## Update: open stack identified (analisis/model/LISENSI.md on their site)
YOLOX (Apache-2.0, player detect), RTMPose (Apache-2.0, pose), TrackNetV3 (MIT) and
WASB-SBDT (MIT) for shuttle, YuNet (MIT, face). refApp1 weights are retrained+encrypted,
but upstream open weights/code can be used for your own build.
Court: badminton 6.1 x 13.4 m, service line 1.98 m from net (matches `core.js` consts).

## Licence note
Their JS carries "all rights reserved, no copying" header. Files in `src/` are reference
for analysis only; do not copy code into `app/`. `app/` is original code.

## app/ status
Done: OPFS upload, 4-click court homography, AR court-line overlay, MP4 export (WebCodecs+mp4-muxer, video only).
Done: pose via YOLOv8n-pose ONNX (Xenova, AGPL upstream) + feet->court metres labels (toggle checkbox).
TODO: swap to RTMPose/YOLOX (Apache) if licence matters, shuttle (TrackNetV3/WASB ONNX), rally/stroke logic, audio passthrough in export.
Run: `cd app && python3 -m http.server 8000` -> http://localhost:8000 (Chrome/Edge).

Shuttle: no ready ONNX found; only TrackNetV3 `.pt` (rahulanantharama/TrackNetV3-badminton). Needs export to ONNX (torch.onnx.export) then plug in as a 2nd session.

## Design v2 (app/index.html)
- Engine registry: pose engines `{load, run(canvas)->people}`; shuttle engines `{load, run(canvas,i)->[x,y]|null}`. New model = one entry.
- Pose options: YOLOv8n-pose (1 stage, AGPL) | RTMPose-s (bukuroo/RTMPose-ONNX) + YOLOX-tiny (cj-mills/yolox-coco-baseline-onnx), Apache-2.0. Default RTMPose.
  Verified in node (onnxruntime-node) on a test photo: YOLOX decode (raw 0-255, strides 8/16/32, 416x544) and RTMPose simcc decode (ratio 2, 192x256, mean/std) give anatomically sane keypoints.
- Shuttle: "off" | TrackNetV3 slot (disabled until an ONNX export exists).
- State machine: idle -> ready -> analysing (progress bar, ETA, cancel) -> analysed -> previewing / exporting.
- Analyse once (every N frames) -> results stored; Preview and Export replay stored results (no re-inference; export is just encode).
- Assumes 30 fps. Not yet tested in a real browser with a real match video.

## Bug fix: "loading models..." stuck
Cause (reproduced in headless Chrome): YOLOX-tiny ONNX has unfolded BatchNormalization; WebGPU EP throws
"BatchNormalization trainingMode is not supported yet" on first run. The error was thrown inside the async Analyse
handler with no catch, so the label never changed. Fix: YOLOX runs on WASM EP; analyse loop now catches and shows "ERROR: ...".
After fix, RTMPose pipeline on a test photo: 3 people found, ~3 s first run (warm-up).

## Tracking + gap filling (app/index.html `track()`)
- After Analyse, detections are matched across analysed frames in court metres (foot point = ankles, else one ankle, else box bottom-centre).
  Greedy nearest-first, gate = min(6, 1+3*dt) m (grows with time since last seen, so a lost player is re-acquired). Max 4 ids; detections > 2 m outside court (spectators) are dropped.
- Each id has a fixed colour used for skeleton, foot dot, label "P#" and mini-map dot.
- Missing frames (and frames skipped by "analyse every N") are linearly interpolated per id (box + 17 keypoints) when the gap <= 1 s; drawn semi-transparent.
- Synthetic test (2 players, swapped order, 6-frame dropout, spectator): ids stay stable, gap filled, spectator ignored.
- Limits: ids can swap if two players cross within ~1 m; long occlusions (>1 s) stay empty; no appearance (colour/shirt) re-identification yet.

## Stats, path tails, live analysis view
- Per player: distance (m) = sum of smoothed court-position steps (5-frame mean, 1.5 cm dead-zone, >12 m/s ignored); calories = MET*kg*h with MET from smoothed speed
  (3.5 / 5 / 6.5 / 8.5 for <0.5 / <1.5 / <2.5 / faster m/s). Rough estimate, weight input in the UI (default 70 kg). Table under the speed block.
  Node test: 1 m/s for 10 s -> 9.9 m, 1.0 kcal@70 kg; standing with +-1 cm jitter -> 0.0 m.
- Mini-map: per-id path of the last 300 frames (10 s), fading with age, clipped to the panel.
- Analysis view: every frame is drawn (no blinking): person boxes + confidence, keypoints coloured by score, TrackNet heatmap of the window's last frame, shuttle path, pose/TrackNet timings.

## Fixes: replay + export
- Export: muxer needs `duration` on every chunk -> VideoFrame now created with `duration`; the first chunk threw, so decoderConfig stayed null and finalize failed with "reading 'colorSpace'". Canvas size forced even (H.264). Export errors shown in status line.
- Replay: the loop used to end when `v.paused` was true on its first callback (fired before play() resolved). Now: await seek(0) + await play(), end detected through 'pause'/'ended' events, guarded by a run id; seek() no longer hangs when already at the target time.
- Verified end-to-end in headless Chrome (synthetic 2 s clip): analyse -> preview x3 (each ends, button = Replay, restarts) -> export OK.

## Stroke detection (v1, rule-based)
`strokesPost()` after `statsPost()` in `track()`. Hit = shuttle image-space velocity jump (angle > ~46 deg or speed ratio > 2.5) with a wrist within 0.5 player-heights; NMS 8 frames.
Type: overhead (wrist above nose) -> Smash (>=150 km/h) / Clear (flight >= 7 m) / Drop; else near net (<2.5 m) -> Lift (>=5 m) / Net shot; Block (fast in, <60% out); Drive (>=90 km/h); else Lift/Shot.
Bad = last hit of rally lands out of court or on hitter's own side (net/short). Labels burned into overlay for 45 frames (1.5 s). Thresholds are untested heuristics (TrackNet untested on real footage); tune with real matches. Not done: technique-quality scoring, audio.

## TODO
- [ ] Per-player play-style summary: one sentence per player (e.g. dominant shots, smash speed, net/out errors, distance covered), shown in the UI and burned into preview/export, EN + FR.
- [ ] Validate the app against refApp2 (hit detection, shot types, smash speed, shuttle tracking) on the refApp2 reference video https://www.reddit.com/r/badminton/comments/1ug5dh1/automated_badminton_rally_analysis/ ; record results and tune thresholds in `strokesPost()`.
