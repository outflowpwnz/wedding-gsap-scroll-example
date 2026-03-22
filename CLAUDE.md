# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file static wedding invitation site (`index.html`). No build step, no dependencies — open directly in a browser.

## Architecture

Everything lives in `index.html`: all CSS is in a `<style>` tag, all JS inline. Google Fonts and GSAP loaded via CDN.

Layout uses a `.page` wrapper (full-width) with `.inner` (max-width: 480px, centered) inside each `<section>` to constrain text on desktop while letting section backgrounds stretch edge-to-edge.

### Design tokens (CSS variables)

| Variable  | Value     | Role               |
| --------- | --------- | ------------------ |
| `--red`   | `#8B1A1A` | Primary / headings |
| `--dark`  | `#2C1810` | Text / footer bg   |
| `--cream` | `#FBF5EC` | Page background    |
| `--gold`  | `#C4882A` | Accents / swatches |
| `--light` | `#E8D5B7` | Hero sub-text      |
| `--paper` | `#F5EBD8` | Alt section bg     |

### Fonts

- **Reznovic** (self-hosted, variable) — section titles (`.section-title`) and hero names (`.hero-title`). Files: `assets/fonts/VFReznovic.woff2` + `.ttf`. Loaded via `@font-face` at top of `<style>`.
- **Philosopher** — fallback for headings; also used for time labels, buttons, misc UI caps (via Google Fonts CDN)
- **Cormorant Garamond** — quotes, RSVP intro (italic serif, Google Fonts CDN)
- **PT Serif** — body text (Google Fonts CDN)

To convert new TTF fonts to WOFF2:
```bash
npx ttf2woff2 < assets/fonts/FontName.ttf > assets/fonts/FontName.woff2
```

### Sections (in order)

1. `.hero` — red background, couple names
2. `.canvas-scene` — scroll-driven frame animation (medallion reveal)
3. `.section` — timeline (`.timeline-item` grid: time / dot / text)
4. `.section.section-alt` — dress code + color swatches
5. `.section` — info/wishes block
6. `.section.section-alt` — RSVP form
7. `.section` — contacts
8. `footer.footer` — dark background closing line

## Canvas scroll animation

The `.canvas-scene` section is a full-viewport pinned block driven by GSAP ScrollTrigger (`anticipatePin: 1`). It preloads 121 WebP frames from `assets/frames/frame_0001.webp … frame_0121.webp` and plays them back as the user scrolls.

### Animation phases (by `self.progress` 0–1)

| Progress    | What happens                                      |
| ----------- | ------------------------------------------------- |
| 0 → 0.15    | Overlay text visible, full cream canvas           |
| 0.15 → 0.30 | Overlay text fades out                            |
| 0.25 → 0.50 | Circular hole grows to medallion size             |
| 0.25 → 0.58 | Ornament bands slide in from sides                |
| 0.35 → 1.0  | Video frames start progressing (frame 0 → 120)    |

### Medallion rendering

`cachedMedR` = `Math.min(canvas.width, canvas.height) * 0.40` (= 80vmin diameter). Cached on resize, not recomputed per tick.

`render()` draws in three steps:
1. `fillRect` — fill entire canvas with cream
2. `drawImage(bitmaps[currentFrame], ...)` — draw pre-scaled frame at medallion position
3. Even-odd overdraw — `rect(0,0,cw,ch)` + `arc(cx,cy,circleR)` filled with cream via `fill('evenodd')`, creating a circular hole that reveals only the medallion area

**Important:** `ctx.clip()` is intentionally avoided — on iOS Safari it forces software stencil compositing. Even-odd overdraw is the correct approach here.

### Frame pre-scaling (`prescaleFrames`)

After load, all 121 frames (720×720 WebP) are pre-scaled to display resolution (~312×312 on iPhone) into offscreen `<canvas>` elements stored in `bitmaps[]`. This makes each `drawImage` a 1:1 GPU blit instead of a scaled texture upload, reducing decoded memory from ~251 MB to ~47 MB. Called once after `syncCanvasSize()` sets `imgDW/DH`.

### Ornament bands

`#orn-band-top` and `#orn-band-bot` are absolutely positioned divs with `background-image: url('assets/orn-band.svg')` (repeat-x). They slide in from opposite sides via `translate3d` during progress 0.25–0.58. Transforms are batched inside the RAF callback alongside `textEl.style.opacity` and `render()`.

### iOS performance notes

- DPR is capped at 1 for touch devices (`isTouchDevice` check)
- `canvas.getContext('2d', { alpha: false })` — opaque context for faster compositing
- `ctx.imageSmoothingQuality = 'low'` — fastest interpolation for downscale
- Body and `.canvas-scene` have **no** `background-image` — tiled radial gradients caused full-document repaint on every scroll tick on iOS
- All RAF mutations (opacity, transforms, canvas draw) are batched in a single `requestAnimationFrame` via `scheduleRender()` with a `rafPending` guard

## Assets

- `site-ref.jpg` — design reference image (original 3-screen mobile layout)
- `assets/video-scroll.mp4` — source video (1440×1440, 24fps, 5s)
- `assets/frames/frame_0001–0121.webp` — pre-extracted frames at 720×720, quality 95
- `assets/orn-band.svg` — ornamental horizontal band, tiled repeat-x on top/bottom of canvas scene
- `assets/ornament-strip.svg` — decorative strip used in hero section

To re-extract frames from the source video:
```bash
for i in $(seq -w 1 121); do
  ffmpeg -i assets/video-scroll.mp4 -vf "scale=720:720" -vframes 1 \
    -filter:v "select=eq(n\,$(( 10#$i - 1 )))" -quality 95 \
    assets/frames/frame_${i}.webp -y -loglevel error
done
# rename to 4-digit padding if needed: frame_001 → frame_0001
```
