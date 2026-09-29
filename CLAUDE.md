# MMM-ThreeBody — context for Claude sessions

Charles's MagicMirror² module: the three-body problem as a long exposure, three stories in turn,
each with a ghost started 10⁻⁶ away, for his hallway mirror, as one page in a rotation of pages.
Split out of MMM-ChaosTheory on 2026-09-28 with its history (it was the `threeBody` simulation
of that module's chaos page).

## Files

- `MMM-ThreeBody.js` — module shell: one canvas plus an HTML caption (equations + live readout,
  updated 2×/s). New sim every `cycleSeconds` (60) and on each `resume()`. Loop: `setTimeout`
  until a frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a sim with
  `resting = true` is polled only every 500 ms; while MagicMirror fades the module out (`hidden`
  is set at the start, `suspend()` comes after), frames draw nothing. `turns: { of, at }`: only
  every nth showing; otherwise the wrapper gets `display: none` and nothing starts.
- `simulations/three-body.js` — the `threeBody` sim on `window.ThreeBodySimulations`. Newton's
  law in a plane, G = 1; adaptive Dormand–Prince 5(4), tolerance 10⁻¹² (error scaled by
  1 + |value|), `System.advance(T)` lands on T exactly. Three `SCENES` (pythagorean, lagrange,
  figure-eight): masses, state, `duration` (sim time), `view`, `rate` (sim units per second:
  54–58 s of real time each), caption note. The ghost = body 1 moved 10⁻⁶ in x. Drawn as a long
  exposure: each frame strokes 6 segments with `lighter`; a body > 1.5 × view out stops being
  drawn; rests at `duration`. Instance `info` carries the scene's caption.
- **Scenes take turns across showings, not within one**: a module-level counter `turn` picks
  `ORDER[turn++ % 3]` (pythagorean → lagrange → figure-eight) each time a `ThreeBody` is
  constructed, i.e. on each `resume()` (and each `cycleSeconds`). `threeBodyScene` pins one.
  The counter is per window, so two instances would share it.
- `simulations/common.js` (`window.ThreeBodyCommon`) — only `sci()` (power-of-ten formatting)
  is used; `FixedClock`, `Trail` and `palette` came along from MMM-ChaosTheory unused.
- UMD-style, so the physics runs in Node: `tests/three-body.test.js` (`node --test`, no
  dependencies): E, p, L kept per scene; Burrau's outcome (4–5 bound, 3 escapes, closest
  approach < 10⁻³); figure-eight and triangle periods; the figure-eight's ghost stays, the
  triangle's ghost breaks by t = 40 while the real one holds, real one broken by t = 72; turns.
- `node_helper.js` — the stats panel (`statsPanel: true`): CPU of Electron and cage, per core,
  temperature, from `/proc`, only while shown.
- `dev/preview.html` — runs the module in a desktop browser (`python3 -m http.server` in the
  repo, then `/dev/preview.html?threeBodyScene=lagrange`).

Timings (real seconds at the default rates): Burrau's third body leaves the view ~45 s in;
Lagrange's ghost triangle breaks within ~30 s, the real one ~50 s in. So a ~60 s page shows each
story through; the mirror's chaos page is 60 s.

MMM-ChaosTheory has the same simulation (`threeBody`): fixes to the simulation belong in both.
The shell (`MMM-ThreeBody.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is shared in
spirit with the sibling modules (MMM-ChaosTheory, MMM-LorenzAttractor, MMM-DoublePendulum,
MMM-FractalBasins, MMM-LogisticMap, MMM-SymmetricIcons, MMM-ChaoticBilliards, MMM-Rule30, and
the non-chaos pages MMM-Atom, MMM-FractalZoom, MMM-Chladni, MMM-SacredGeometry, MMM-Tilings,
MMM-PlanetsDance, MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck): a fix there probably belongs
in the siblings too.

On the mirror this simulation runs as part of MMM-ChaosTheory's chaos page (60 s, one
simulation per showing), not as this module.

## Measured cost on the Pi

900², 20 fps, Electron + cage, over a 60 s showing (Burrau's problem and Lagrange's triangle;
the figure-eight not measured): 76% of a core. Hidden: 0.3% (baseline 0.2%). MMM-ChaosTheory's
README records 24 fps achieved for it, above the 20 fps cap: probably a measuring artefact, so
the README here leaves it out; measure again if it matters.

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame).
- So: draw incrementally (long-exposure trails), keep each frame's changes spatially compact,
  and rest when the picture is static. Line width, opacity, `rAF` vs timer made no difference.
- MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be set
  that way; use `debugStats: true` and a `grim` screenshot to see fps on the Pi.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. **Measure, don't guess**: on the Pi,
  `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS over 60 s. Record before/after numbers
  in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). `suspend()` and
  `resume()` must fire on page changes, or the loop burns CPU 24/7.

## Deploying and testing

- This repo is public so the Pi can `git clone`/`git pull` without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-ThreeBody`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: run it in a desktop browser (`dev/preview.html`), then confirm performance
  on the Pi.
- Commit as Charles's GitHub noreply address (set in this repo's git config).
