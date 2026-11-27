# Badminton Analysis Tracker

Client-only badminton rally analyser: pose, shuttle tracking, stats, shot detection, AR overlay and export. Runs fully in the browser (WebGPU, WebCodecs).

## Run
`cd app && python3 -m http.server 8000`, open in Chrome/Edge.

## Use
1. Upload a video (kept in browser storage across refresh).
2. Court corners are auto-detected when sure; otherwise click far-left, far-right, near-right, near-left.
3. Analyse, then Preview or Export.

Built-in overlays: skeleton, meter position, smash map (yellow arrows, numbered per rally). Features: reaction speed, court occupation.

See `SPEC.md` for design notes and `AGENTS.md` for developer notes. `src/` is third-party reference code, not to be copied.
