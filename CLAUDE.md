# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

The repo holds browser games, one per folder (`<folder>/<file>.html`). The root `index.html` ("태오 오락실") is a launcher page that links to each game. Right now there is one game: `shooting/play.html` ("뿅뿅 슈터"), a vertical neon shoot-'em-up for 1 or 2 players with 80 stages. The UI text and code comments are in Korean.

**Adding a game:** create its folder and HTML file, then add an entry to the `GAMES` array in `index.html` (`path`, `title`, `icon`, `color`, `desc`, `tags`). There is no build step, so the launcher can't discover folders on its own; a game that isn't listed in `GAMES` won't appear.

There is no build step, package manager, linter, or test suite. Each game is a single self-contained HTML file. Its only outside dependencies are three.js r0.160.0 (loaded from jsDelivr through an `<script type="importmap">`) and Google Fonts.

## Running

ES module imports don't work over `file://`, so serve the folder over HTTP:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000/` (the launcher) or `http://localhost:8000/shooting/play.html` directly. To check a change, load the page in Playwright (Chromium is pre-installed) and look at the console for errors. WebGL has to work for the game to render.

## Architecture of `shooting/play.html`

Everything lives in one `<script type="module">`, in this order:

1. **Audio**: everything is synthesized with WebAudio, and there are no audio files. `laser`, `beep` and `noise` make sound effects. A step sequencer (`BGM`, `playStep`, `musicTick`) plays procedural music and switches between `calm`, `play` and `boss` tracks depending on `state`. `initAudio()` must run from a user gesture.
2. **Game state**: these are module-level `let` globals, not objects: `state` (`'title' | 'select' | 'play' | 'pause' | 'over' | 'allclear'`), `mode` (1 or 2 players), and the entity arrays `players`, `pb` (player bullets), `eb` (enemy bullets), `en` (enemies, including the boss), `items`, `parts`, `rings`, `texts`. `reset()` sets them all back to their starting values.
3. **Stage system**: there are 8 `SECTORS` with 10 stages each. Stage 5 of each sector is an elite stage and stage 10 is a boss stage. `buildStage(n)` uses a seeded RNG (`mulberry`) to build a deterministic, time-sorted `script` of spawn events from named patterns in `addPattern`. Within a stage, `stagePhase` goes `intro → fight → clear`. Difficulty `level` comes from the stage number.
4. **Coordinates**: game logic runs in a fixed 450×800 logical space (`W`, `H`). The 3D scene maps it onto the ground plane with `LX(x)` and `LZ(y)`, which divide by 10, at height `FLY`. `toScreen` projects 3D back to overlay coordinates. `toField` raycasts pointer input onto the flight plane.
5. **Rendering uses two stacked canvases**:
   - `#gl` holds the three.js scene, rendered through an `EffectComposer` with `UnrealBloomPass`. Entities get lazily created Object3Ds that are kept in step with the logic arrays by `syncEnemies`, `syncItems` and `syncShips`, which store the mesh as `e.obj`. Glow particles are one big `Points` buffer filled every frame through `G()`/`GL()` (up to `MAXG` points). Lasers and debris use `InstancedMesh`. Materials are cached by color (`edgeMat`, `basicMat`), and the neon look comes from `addNeon`, which pairs a mesh with its edge lines.
   - `#c` is a 2D canvas overlay that draws the HUD, menus, banners and buttons in `drawUI()` and its `draw*` helpers. It uses the same logical 450×800 coordinates, scaled by `dpr`.
6. **Main loop**: `loop()` calls `update(dt)`, then `render3D()`, then `drawUI()`. `dt` is capped at 0.033. `hitStop` slows time briefly for impact feel.

## Controls and persistence

- **Home button**: the title screen has a ⌂ button in the top-left (`HOME_BTN`), and Esc on the title screen also works. Both go to `../index.html` through `goHome()`. New games should provide the same way back to the launcher.
- **Controls**: pointer drag (multi-touch through the `drags` map), WASD for 1P, and arrow keys (1P in solo mode, 2P in two-player mode). P or Esc pauses, M mutes, B toggles music, and Enter or Space confirms.
- **localStorage keys**: `ppyong_best` and `ppyong_best2` (high scores per mode), `ppyong_stage1` and `ppyong_stage2` (highest unlocked stage per mode, up to 81), and `ppyong_music`. Every storage access is wrapped in try/catch. Keep it that way.
