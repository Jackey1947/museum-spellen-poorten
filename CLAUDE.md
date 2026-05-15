# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Stadspoorten van Maastricht — a drag-and-drop museum game where the player matches the seven historic city gates to their locations on an 1840–1845 cadastral map. UI text is in Dutch. Live build is published via GitHub Pages from `main` at https://jackey1947.github.io/museum-spellen-poorten/.

## Running

There is no build, no package manager, no test suite. The whole game is a single self-contained file: `index.html`.

- Local dev: open `index.html` directly in a browser (`open index.html` on macOS). No server required — the historic map is embedded as a base64 image so the file works fully offline.
- Deploy: pushing to `main` updates the GitHub Pages site.

When iterating on layout, drag behavior, or drop-zone positions, reload the file in the browser and exercise both desktop drag-and-drop and a touch path (e.g. browser devtools touch emulation) — they are independent code paths (see Architecture).

## Architecture

The entire app lives in `index.html`:

- `<style>` (top of file) — all CSS, including pulsing-dot animation (`@keyframes expand`), wrong-answer shake (`@keyframes shake`), and a `@media (max-width: 900px)` breakpoint that switches the layout from side-by-side to stacked.
- `<body>` — `#topbar` (score + attempts), `#progress-bar`, `#map-col` with seven `.dz` drop-zones positioned absolutely as `left%/top%` on the map image, and `#sidebar` containing the draggable cards and the `#msg` feedback box. A floating `#ghost` element is used as the drag preview on touch.
- `<script>` (bottom of file) — vanilla JS, no framework, uses `var` and direct DOM APIs.

### The `gates` array is the source of truth

`var gates = [...]` (near line 210 of `index.html`) defines all seven gates with `{id, name, hint, num}`. The `id` is the join key: it matches `data-gate="..."` on each `.dz` element in the HTML, and `card-<id>` ids on the dynamically generated sidebar cards. Adding, removing, or renaming a gate means updating *both* the `gates` array and the matching `.dz` element in the HTML.

### Drop-zone positions

`.dz` positions are inline `left:%;top:%` on the `.dz` elements in the HTML. The canonical values live in `Posities-poorten.txt` — when the user provides updated coordinates, apply them to the corresponding `.dz` inline styles in `index.html`. The map image is rendered with `object-fit: contain; object-position: center top`, so percentages are relative to the image's rendered box, not the viewport.

### Two parallel drag implementations

Desktop and touch drag are deliberately separate:

- Desktop: native HTML5 drag-and-drop. `buildCards()` attaches `dragstart`/`dragend` to each card and stores the active gate in module-level `dragId`. `buildDZ()` attaches `dragover`/`drop` to each `.dz`, and `drop` calls `check(dz, dragId)`.
- Touch: manual handlers `onTS`/`onTM`/`onTE` track `tCard`/`tGate`, move the `#ghost` preview with the finger, and on `touchend` do a manual `getBoundingClientRect()` hit-test against all `.dz` elements before calling `check()`.

Both paths funnel into the same `check(dz, gid)` function, which is where scoring, progress bar, and win logic live. Score formula: `Math.max(10, 100 - Math.max(0, attempts-7)*8)` — the first 7 attempts are "free", each extra attempt costs 8 points, with a floor of 10.

### Reset

`resetGame()` zeroes state, clears `correct`/`wrong`/`over` classes from drop-zones, and calls `buildCards()` again (which reshuffles the sidebar cards via `gates.slice().sort(() => Math.random()-.5)`). It does not rebuild drop-zones since their bindings persist.
