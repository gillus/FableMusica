# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Suono** is a visual music instrument: the user paints particle trails with the mouse or a finger, and the trails make sound. The whole app is `index.html`, a single self-contained file with inline CSS and JS. It uses no libraries, no build step, no package manager and no tests. It uses only the canvas 2D API and the Web Audio API.

- The UI text, code comments, commit messages and README are in **Italian**. Keep them in Italian.
- It is deployed with GitHub Pages from the root of `main`: https://gillus.github.io/FableMusica/. `.nojekyll` turns Jekyll off. `suono.html` is only a meta-refresh redirect to `./` so old links keep working. Don't put app code in it.

## Running

Open `index.html` directly in a browser, or serve the folder (`python3 -m http.server`) and open it. Audio starts only after the user clicks the `#start` overlay (the browser autoplay policy), so `ctx` stays `null` until then. Most handlers return early while `ctx` is null.

The code has no lint and no test commands. To verify a change, load the page and interact with it. Check the desktop layout and the mobile layout (≤720px). `isMobile` is evaluated **once at load**, so reload after resizing.

## Architecture (inside the single `<script>` IIFE)

**Palettes drive everything.** Each entry in `PALETTES` sets the visuals (`bg`, `fade`, `blend`, `stamp`, `colors`, `voiceColors`), the synth patch (`sound`, which also holds the default reverb and echo) and the drum kit and 16-step patterns (`drums`: `x` is a full hit, `o` a soft hit, `.` a rest). The particle shape generators are chosen by `palKey` in `spawnStroke` and `burst`: `kandinskyBit`, `monetDab`, `mondrianStroke`. Adding a palette touches the `PALETTES` entry, a `#palette` button (with `--sw` for the swatch color), those `palKey` branches, the digit keyboard shortcut, and the README.

**Two-canvas rendering.** `paint` is an offscreen layer. Particles are stamped onto it at `PAL.stamp` alpha, and each frame it fades toward `PAL.bg`: strongly every 3rd frame, lightly otherwise, to avoid ghost residue. The visible canvas `c` is `paint` plus the same live particles drawn again at full alpha. `resize()` keeps the painting by copying it through a snapshot.

**Beat clock.** Time is measured in beats: `beatAt(t)` and `timeOfBeat(b)` convert between beats and seconds, using `loopOrigin` and `bpm`. `setBpm` rebases `loopOrigin` so the current beat stays continuous. `scheduler()` runs every 30 ms with a `LOOKAHEAD` of 0.12 s. It schedules drum steps (sixteenth notes, so `step/4` gives beats) and loop-voice events at exact `AudioContext` times. If the scheduler stalls for more than 1 s (for example in a hidden tab), `resync()` re-aligns every cursor.

**Recording and voices.**
- While `rec` is active, `handle()` records samples in normalized coordinates: `nx`/`ny` in 0..1, and `dx`/`dy` as fractions of W/H. Each sample carries its beat position inside the loop.
- Recording finalizes after exactly one loop, either from `recordSample` or from `frame()`.
- The pitch of a recorded note is recomputed at playback (`midiOf(noteIndex(e.nx))`), so voices follow the current scale. The color comes from the current palette's `voiceColors`.
- Each voice keeps a playback cursor (`cycle`, `idx`). Events past the loop length, which happens after `bars` is reduced, are skipped.
- A note's visuals are pushed to `visualQueue` and spawned in `drainVisuals()` when audio time reaches them, so what you see matches what you hear.

**Shared stroke state.** The `st` argument of `spawnStroke` and `mondrianStroke` is the pointer state object for live input, or the voice object for playback. Both carry `cell` and `cellT` for Mondrian's per-cell throttle.

**Audio graph.** Each note creates its own oscillator pair, filter, envelope and panner, and feeds `instBus`. `instBus` goes to `master`, to the convolver reverb, and to the tempo-synced delay (`0.75 * 60 / bpm`, which also feeds the reverb). The drums go to `drumBus`. Everything passes through `master`, then a compressor, then the destination. `activeNotes` caps polyphony at about 36 notes.

**Saving images.** `saveImage()` uses `window.claude.use('downloads')` when the page runs as a claude.ai artifact. Everywhere else it falls back to a blob `<a download>` link. Keep both paths.
