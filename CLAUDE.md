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

The `.canvas-scene` section is a full-viewport pinned block driven by GSAP ScrollTrigger. It preloads 121 WebP frames from `assets/frames/frame_0001.webp … frame_0121.webp` and plays them back as the user scrolls.

### Animation phases (by `self.progress` 0–1)

| Progress    | What happens                                      |
| ----------- | ------------------------------------------------- |
| 0 → 0.15    | Overlay text visible, full cream canvas           |
| 0.15 → 0.30 | Overlay text fades out                            |
| 0.25 → 0.50 | Circular hole grows to medallion size             |
| 0.35 → 1.0  | Video frames start progressing (frame 0 → 120)    |

### Medallion rendering

`medallionR()` = `Math.min(canvas.width, canvas.height) * 0.40` (= 80vmin diameter).

`render()` draws in two passes:
1. Video frame clipped to the medallion circle via `ctx.clip()` — prevents white video background from showing
2. Cream overlay drawn with an evenodd hole cut at `min(currentCircleR, medallionR)` — keeps the cream surround permanent

The `#medallion-ornaments` SVG (double gold ring, `100vmin × 100vmin`, `viewBox="0 0 100 100"`) sits over the canvas at z-index 3 and fades in with `holeT` from JS.

## Assets

- `site-ref.jpg` — design reference image (original 3-screen mobile layout)
- `assets/video-scroll.mp4` — source video (1440×1440, 24fps, 5s)
- `assets/frames/frame_0001–0121.webp` — pre-extracted frames at 720×720, quality 95

To re-extract frames from the source video:
```bash
for i in $(seq -w 1 121); do
  ffmpeg -i assets/video-scroll.mp4 -vf "scale=720:720" -vframes 1 \
    -filter:v "select=eq(n\,$(( 10#$i - 1 )))" -quality 95 \
    assets/frames/frame_${i}.webp -y -loglevel error
done
# rename to 4-digit padding if needed: frame_001 → frame_0001
```
