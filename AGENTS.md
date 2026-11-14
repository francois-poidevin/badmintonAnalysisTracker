# AGENTS.md

Client-only badminton rally analyser (own implementation, inspired by refApp1). No build, no package.json, no git repo.

## Layout
- `app/index.html`: the whole app (single file, one inline `<script type="module">`). Edit this.
- `src/`: **downloaded refApp1 JS, reference only.** Header says "all rights reserved, no copying". Never copy code from it into `app/`; their `.vsm` models are AES-encrypted + domain-locked, do not try to decrypt/fetch them.
- `SPEC.md`: findings, design decisions, bug post-mortems. Append new decisions/fixes there.

## Run / verify
- Serve: `cd app && python3 -m http.server 8000`, open in Chrome/Edge (WebGPU, WebCodecs needed). `file://` will not work.
- Syntax check after every edit (no linter exists):
  `python3 -c "import re;open('/tmp/opencode/t.mjs','w').write(re.search(r'<script type=\"module\">(.*)</script>',open('app/index.html').read(),re.S).group(1))" && node --check /tmp/opencode/t.mjs`
- Browser-level test that worked: headless `google-chrome --headless=new --no-sandbox --enable-unsafe-webgpu --autoplay-policy=no-user-gesture-required --enable-logging=stderr --user-data-dir=<fresh dir>` loading a temporary `app/t.html` that embeds `index.html` in an iframe, builds a webm with MediaRecorder, drives the buttons, and `console.log('LOG ...')`s status. Read output with `grep CONSOLE`. Delete `t.html` afterwards. Run chrome in background + `sleep` (tool timeout 120 s); never `pkill -f` a pattern that matches your own shell command.
- Pure logic (`track`, `statsPost`, `shuttlePost`) can be tested in node by slicing the script text and `new Function(...)`; `onnxruntime-node`/`sharp` in `/tmp/opencode` were used to validate model I/O.

## Architecture (app/index.html)
- Flow: state machine `idle -> ready -> analysing -> analysed -> previewing/exporting` (`setState`). Analyse stores per-frame results once; Preview and Export only replay stored results (no inference).
- Engine registry (`POSE`, `SHUT`): pose engine `{load, run(canvas)->[{box,kp[17],cf}]}`; shuttle engine `{load, background, frame, run(8 frames, bg)}`. New model = new entry.
- Models are fetched at runtime from Hugging Face via onnxruntime-web 1.22 (CDN). RTMPose-s `bukuroo/RTMPose-ONNX`, YOLOX-tiny `cj-mills/yolox-coco-baseline-onnx`, YOLOv8n-pose `Xenova/yolov8n-pose`, TrackNetV3 `jinergenkai/badminton-analysis-web` (fp32, input `(1,32,288,512)` = median bg + 8 frames RGB/255 + zero pad).
- Post-processing chain in `track()`: id tracking in court metres -> dense interpolation (gap <= 1 s) -> `shuttlePost` (km/h) -> `statsPost` (distance, kcal). Overlay (`overlay`, `tail`, `minimap`) is drawn on the canvas itself, so it is burned into preview and export.
- Court homography from 4 clicked corners, order matters: far-left, far-right, near-right, near-left. Metres: 6.1 x 13.4.
- Assumes 30 fps (`FPS`); seeking frame by frame (`seek(i/FPS)`).

## Gotchas learned the hard way
- YOLOX-tiny ONNX **fails on the WebGPU EP** (unfolded BatchNorm); it is pinned to `['wasm']`. Errors inside the async Analyse handler are swallowed unless caught: keep the try/catch that writes `ERROR:` to the status line.
- YOLOX decode: raw output, grid decode with strides 8/16/32 on a 416x544 input, raw 0-255 pixels.
- mp4-muxer needs `duration` on every `VideoFrame`; canvas size must be even (H.264). Otherwise `addVideoChunkRaw ... duration` / `reading 'colorSpace'`.
- Preview end must be detected with `pause`/`ended` events after `await play()`; checking `v.paused` inside the rVFC callback ends replays instantly.
- Draw the overlay on every frame while analysing (blinking otherwise); `liveDraw` redraws the video first.
- Shuttle speed is a ground-plane approximation; kcal is a rough MET model; both are labelled approximate in the UI.
- Known TODO: stroke/rally detection, audio in export, track id swaps when players cross. TrackNet detection is untested on real match footage.
