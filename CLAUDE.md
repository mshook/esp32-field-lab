# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A static, browser-only ESP32 teaching app (16 lessons + a breadboard "Build" mode) using Three.js. Plain ES modules in `dist/` — **there is no build step**; `dist/` *is* the source and is what GitHub Pages deploys. Three.js and fonts are vendored in `dist/vendor/` and `dist/fonts/`. No backend; all persistence is `localStorage`.

## Commands

```sh
npm start                     # python3 -m http.server 4173 -d dist (must serve over HTTP; modules won't load from file://)
npm run check                 # node --check on every dist module (add new modules to this list)
npm run test:circuit          # pure-Node circuit-rule + walkthrough tests (fast, no browser)
npm run test:build            # test:circuit + Build-mode Playwright suite
npm test                      # everything, including all lesson/guide/viewport browser suites
node tests/build-circuit.cjs  # run any single test file directly
```

Browser tests need `npx playwright install --with-deps chromium`, or set `CHROMIUM_EXECUTABLE` to an existing Chromium. CI (`.github/workflows/pages.yml`) runs only `check` and `test:circuit` before deploying on push to `main`.

Test env knobs: `QA_OUTPUT` (screenshot dir; defaults under `/tmp/esp32-*-qa`), `QA_SKIP_LESSONS=1` (skip per-lesson loops in `browser.cjs`/`guides.cjs`), and for `tests/build-browser.cjs`: `QA_CASE=` one of `wire-retry|editing|camera|default-wiring|mobile-drag|touch-endpoint|guidance` to run a single scenario, `QA_VIEW=2d`, `QA_SKIP_PICKER=1`. Browser tests bind fixed ports 4181, 4184, 4185, 4194.

Tests are CommonJS (`.cjs`) that dynamically `import()` the ESM modules from `dist/`; tests use a tiny inline `test(name, fn)` + `node:assert/strict`, no framework.

## Architecture

- `app.js` is the entry: lesson navigation, tabs, quizzes, progress, and hash routing. `#<lessonId>` selects a lesson; `#build-<projectId>` swaps `#lesson-main` for `#build-root` and calls `mountBuild()`.
- `scene.js` exports `createScene(container, callbacks)` — a single large closure owning all procedural 3D models, camera, highlights, interactions, and the Build-mode 3D rendering. Both lessons and Build mode use it.
- Lesson content lives in data modules: `lessons.js` (lessons, sketches, quiz, `parts` reference), `guides.js` (visual connection tours), `diagrams.js` (SVG schematics). Adding a lesson means adding data here; tests assert counts (16 lessons, 68 guide steps, 51 pinout rows), so update those when content changes.

### Build mode layering (keep the split)

- `build-circuit.js` — pure, DOM-free model: endpoint ids (`bb:<col><row>` breadboard, `rail:…`, `pin:<gpio|GND|GND2|3V3…>` with aliases via `canonicalEndpoint`), nets, part footprints/`placement`, occupancy, wire resolution (occupied contact → free contact on the same net), `validateCircuit` (returns `{ok, code, …}`), `simulationState`, `connectionPlan`, `repairPlan`. All 430 contacts are enumerated by `allEndpoints()`.
- `build-projects.js` — project catalog (`blink`, `button`, `potentiometer`): parts, default placements, control, and Arduino code pulled from the matching lesson. Project ids must equal lesson ids.
- `build-walkthroughs.js` — `getWalkthrough(state)` produces guided steps derived from the learner's *actual* layout, not a fixed script.
- `build.js` — `mountBuild()` UI/editor state (modes, drafting wires, undo, hints, check/run) wiring the pure model to `scene.js`. Drafts persist per project under `field-lab-build-<id>-v1` (blink uses the legacy `field-lab-build-blink-v1` key) and guide prefs under `field-lab-build-guide-<id>-v1`.
- `build-project-picker.js` — standalone styled picker component.

Circuit rules belong in `build-circuit.js` and are covered by `tests/build-circuit.cjs`; UI behavior is covered by `tests/build-browser.cjs`.

### Test instrumentation contract

Browser tests intercept `scene.js` and inject observation globals (e.g. `__labScene`) just before the exact line `  externalSetup('board');resize();`, asserting it appears exactly once. Don't rename/reformat that line without updating `tests/browser.cjs`, `tests/guides.cjs` and `tests/build-browser.cjs`. The injected code reads closure-internal names from `scene.js` (`world`, `camera`, `renderer`, `meshes`, `circuit`, `board`, `hit`, `interaction`, `buildWireHandles`, `buildCopper`, `buildHighlightTargets`, …), so renaming those also breaks tests. Production code must not expose test globals.

## Conventions

- Code is written in a dense, compact style (long single lines, short identifiers, minimal whitespace) in most `dist/*.js` files and tests; match the surrounding file's style rather than reformatting.
- Asset paths must stay relative — the site is served from a project subpath (`/esp32-field-lab/`, exercised by `tests/viewport.cjs`).
- Target hardware is the original 30-pin ESP32 DevKit V1 / ESP32-WROOM-32 with Arduino-ESP32 3.x; GPIO is 3.3 V. Keep pin names and sketches consistent with that board.
- Lesson output controls rewrite constants in the displayed sketch (e.g. `const int INTERVAL_MS = …;`); tests check these exact declarations.
