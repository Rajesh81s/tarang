# Latency Design Notes — Tarang Air Harp

## Pipeline Latency Budget

The target is **< 50ms** from finger movement → sound output.

| Stage | Budget | Notes |
|---|---|---|
| Camera frame capture | ~16ms | 60fps camera; could be 33ms at 30fps |
| MediaPipe inference | ~10-15ms | GPU-accelerated, float16 model |
| Gesture + pluck detection | ~1ms | Pure math, segment intersection |
| Audio buffer creation | ~0ms | Pre-cached Karplus-Strong buffers |
| AudioContext scheduling | ~3-5ms | Web Audio API render-ahead |
| **Total** | **~30-35ms** | Well within 50ms budget |

## Key Optimization Decisions

### 1. Karplus-Strong Pre-caching
Notes are generated once and cached. Each subsequent play of the same MIDI note
reuses the pre-generated buffer — zero synthesis latency after first play.

### 2. Landmark Smoothing
We use exponential smoothing with factor 0.35. This introduces ~1 frame of
visual latency but significantly reduces jitter from noisy hand tracking.
The smoothing does NOT affect pluck detection timing — we use the raw frame-to-frame
delta for collision detection.

### 3. Pluck Cooldown
130ms cooldown prevents double-triggers from a single swipe. This is short enough
that intentional rapid plucking (up to ~7.7 plucks/second per string) still works.

### 4. Web Audio API Buffer Size
We don't set an explicit buffer size — the browser picks the smallest safe value
(typically 256 or 512 samples = 5-11ms at 48kHz). This keeps output latency minimal.

### 5. requestAnimationFrame
We use rAF instead of setInterval for the main loop. This syncs our processing
to the display refresh rate (typically 60Hz), avoids unnecessary work, and
lets the browser optimize GPU scheduling.

## Measurement

To measure actual latency:
1. Use Chrome DevTools Performance tab
2. Record a pluck gesture
3. Measure from the frame where finger crosses string to when AudioContext.currentTime
   shows the note start

Expected: 25-40ms on modern hardware.
