# Robot Rumble — Revised Edition: Audit, Changes and Tests

Game by Carlos Talegón. Revised so it follows the "rules for a great simple 2D HTML
game" checklist, with minimal, conservative changes: the game, its content, its
50 robot designs, its weapons, its music and its hot-seat multiplayer (2 teams of
1–10, or free-for-all with 2–10 robots) are all still Carlos's game. Every fix in
`index.html` is marked with a short `// RULE:` comment so the diff can be read as
a lesson.

---

## A. Audit findings

### Design rules

| Rule | Verdict | Evidence |
|---|---|---|
| One clear core loop | ✅ Pass | Aim → charge → fire → terrain/robots react → next turn. One main verb (aim & shoot) on a turn. |
| Goal & threat always visible | ⚠️ Mostly | Team life bars, weapon bar, current robot's "Here!" bubble and rising water are always on screen — but **wind was invisible** (see fix 1), and team life was a bar with no number in Teams mode (fix 2). |
| Instant, juicy feedback | ✅ Pass | Particles, floating text, screen shake, speech bubbles ("Ouch!"), synthesised beeps, hit flashes, crown for the leading robot. |
| Readability | ✅ Pass | Robots are DOM/SVG actors with distinct designs and names; teams have names as well as colours; weapon buttons are labelled with text. |
| Fair difficulty ramp (one variable at a time) | N/A by design | There are no levels: difficulty is the other players. The optional sea-level rise is the ramp, and it is player-chosen (No / Slowly / Normally / Quickly) — a good pattern. Left as designed. |
| Fast failure / instant restart | ✅ Pass | PLAY AGAIN on the game-over screen and RESTART in the pause menu both start a new match in one click, with no menus in between. (One deliberate exception, left unchanged: pausing fades the music over ~2 s before the pause lands — see §C.) |
| Controls fit the device; multiplayer without interference | ✅ Pass (by design) | Hot-seat turns: only one robot acts at a time, so 2+ players sharing one keyboard can never fight over keys. The game states plainly that a desktop keyboard is required; the mouse is only used for menus/weapon choice. |
| Fair randomness | ⚠️ One real flaw | Spawn slots are shuffled, the first team is random, supply crates drop at random — all symmetric and fair. But **wind (which pushes rockets) was chosen randomly each turn and never shown**, so a shot could be pushed by an invisible force. Fixed by fix 1. |
| Score / a reason to replay | ⚠️ Weak | A winner is declared, but nothing was remembered between matches. Fixed by fix 3 (local win/match tally). |
| Accessibility | ⚠️ Gaps | Pause was only on `P`; screen shake and CSS animation ignored `prefers-reduced-motion`. Fixed by fixes 4 and 5. Team life is now also numeric (fix 2). Focus outlines and aria labels were already present in menus. |

### Technical (HTML) rules

| Rule | Verdict | Evidence |
|---|---|---|
| Canvas for the game, HTML for menus | ✅ Pass | Battle renders on `<canvas>`; settings, robot chooser, instructions and footer tools are HTML/CSS. Robots are DOM/SVG actors layered over the canvas — unusual, but deliberate (it gives the tooltip/hover behaviour) and bounded (≤20 robots). |
| `requestAnimationFrame` **with clamped delta time**; speeds in units/second | ❌ **Fail — the big one** | `function loop() { update(); draw(); requestAnimationFrame(loop); }` passed no time at all. Every speed, timer and fuse in the game is counted in frames, so on a 120 Hz screen the game would run at double speed, and on a slow screen in slow motion. Fixed by fix 6. |
| `update()` / `render()` separation | ✅ Pass | `update()` and `draw()` were already separate functions. Kept. |
| Explicit state machine | ⚠️ Implicit | `state` (`presentation`/`menu`/`play`/`over`) + `paused` boolean already behaved as a state machine and handlers already checked it, but it was nowhere written down. Documented in code by fix 7 (no behaviour change — renaming states would have meant touching hundreds of comparisons for zero benefit). |
| All tuning in one CONFIG block | ⚠️ Partial | Many tuning numbers were already named constants grouped with their systems (`MAX_ENERGY`, `MAGNET_RANGE`, weapon data…). The new loop/effects tuning lives in one new `CONFIG` block (fix 6); existing constants were left where they are rather than moved (moving them risks breaking references and makes the diff unreadable). |
| Simplest-shape collisions | ✅ Pass | Terrain is a pixel mask, projectiles step in sub-steps and test points/radius — appropriate for destructible terrain. No change. |
| Procedural vs asset graphics | ✅ Pass | Robots/terrain/effects are drawn in code; the PNG is the title screen and the MP3s are music — sensible asset use. One asset was broken: see fix 8. |
| Responsive scaling + sharpness (devicePixelRatio) | ✅ Pass (with note) | The stage scales to the window keeping the 2:1 aspect (`width:min(100vw, …)`), and the canvas backing store is a fixed 2400×1200 — i.e. the world is always rendered at 1.5625× its 1536×768 logical size, so it is supersampled (sharp) on ordinary screens and phones. On a very large high-DPI monitor it can upscale slightly. Making the backing store dynamic would require re-plumbing every HUD coordinate, so it was left unchanged (see §C). |
| Performance: no per-frame allocation / DOM churn in the loop | ⚠️ Partial | Particle count was **uncapped** (a big explosion chain could keep adding particles) — capped by fix 9. Remaining, accepted: arrays are rebuilt with `.filter()` each step (bounded, small collections), the sky gradient is recreated per frame (normal canvas practice), and `syncActors()` writes DOM styles for ≤20 robot actors per frame (that is the game's actor design). See §C. |
| Sound starts after a user gesture; mute available | ⚠️ Mostly | Music only starts from the presentation click / Start button, and there are footer mute + volume controls for sound and music. But the Web Audio context in `audio()` could be *created* by any early `beep()` call before a gesture. Guarded by fix 10. |
| Saves stay local (`localStorage` only) | ✅ Pass | The original saved nothing at all. The new match tally (fix 3) uses `localStorage` only. |

### Asset bug found while auditing

- The code loads `Triomphe des Alliés.mp3`, but the file as extracted from the
  zip is named `Triomphe des Alli#U00e9s.mp3` (the `é` was mangled) — so the
  intro theme would 404 / fail to load. Fixed by fix 8.

---

## B. Changes made (each maps to a rule above)

1. **Wind is now shown in the HUD** — a small pill at top-centre during play:
   `WIND: CALM`, `WIND: ◀ 7` or `WIND: ▶ 7`, updated every turn.
   *Rule: goal/threat always visible + fair randomness — never hide a force that affects the player's shot.*
2. **Team life is also a number** — Teams-mode HUD now reads e.g.
   `Blue Team  3/3  ·  247 HP` next to the bar.
   *Rule: no information by colour/bar alone (accessibility).*
3. **Local match history** — when a match ends, the result is recorded in
   `localStorage` (`robotRumbleMatchHistoryV1`: matches played + wins per side),
   and the game-over screen shows the tally under the buttons.
   *Rule: a score / reason to replay; saves stay local.* Loaded once into memory — never read per frame.
4. **Escape also pauses** — `Esc` works everywhere `P` did (pause/resume).
   *Rule: accessibility — pause on the key players expect.*
5. **Reduced motion respected** — if the player has `prefers-reduced-motion: reduce`,
   screen shake is skipped in `draw()` and a CSS media query stops the decorative
   animations (presentation prompt pulse, actor animations).
   *Rule: accessibility / reduced motion.*
6. **Delta-time game loop (the main fix)** — `loop()` now receives the rAF
   timestamp, computes a **clamped** delta (`CONFIG.maxFrameTime` = 0.25 s), and
   runs the simulation in **fixed 1/60 s steps** from an accumulator
   (`CONFIG.fixedStep`), capped per frame (`CONFIG.maxStepsPerFrame` = 5, backlog
   dropped instead of fast-forwarding). This is the conservative way to satisfy
   "speeds in units per second" here: all of the game's hundreds of tuned
   per-frame numbers (gravity 0.2/step, fuses of 180 steps, charge 0.016/step…)
   keep exactly their tuned values, but real-world speed is now identical at
   30 Hz, 60 Hz and 120 Hz, and a background-tab stall can no longer teleport
   the game. `update()` and `draw()` themselves are untouched.
   *Rule: rAF with clamped delta time; update/render separation preserved.*
7. **The state machine is written down** — a comment block above `state`
   documents every state and transition (`presentation → menu → play ⇄ paused →
   over → play/menu`), and notes that input handlers gate on it.
   *Rule: explicit state machine.* Documentation only, zero behaviour change.
8. **Intro theme filename fixed** — the file in this folder is correctly named
   `Triomphe des Alliés.mp3`, and the code now references it percent-encoded
   (`Triomphe des Alli%C3%A9s.mp3`) so it survives zips and servers that mangle `é`.
   *Rule: asset paths must match the file on disk; keep relative paths working.*
9. **Particle cap** — `CONFIG.particleCap` (420): `addParts()` skips excess
   sparks and `stepFx()` trims any other particle source to the cap.
   *Rule: performance — bounded per-frame work.*
10. **Audio starts only after a gesture** — `audio()` now returns `null` (and
    creates no `AudioContext`) until the player has clicked or pressed a key;
    all `beep()` callers already handled a null context.
    *Rule: sound after user gesture (autoplay policy).*
11. **A tiny test hook** — `window.__robotRumble` exposes read-only
    `state` / `paused` / `frame` getters. Not used by gameplay; it exists so the
    game can be verified by automated checks (used by the tests in §D).

Also in this folder: `README.md` (Carlos's, unchanged) and all original assets,
copied with the one filename correction in fix 8.

---

## C. Deliberately left unchanged (and why)

- **The ~2-second music fade before pause lands.** Pressing P cross-fades the
  gameplay/pause music over 2 s and only then toggles `paused`, so the game
  keeps running briefly after the keypress. Making pause instant would mean
  rewriting the pause-music system and losing the cross-fade Carlos built.
  It is a real responsiveness trade-off, documented here rather than silently
  "fixed" — an easy follow-up if Carlos prefers instant pause.
- **Fixed 2400×1200 canvas backing store** (see the DPR note in §A). Already
  supersampled for phones/laptops; dynamic resizing risks every HUD coordinate.
- **Per-step `.filter()` array rebuilds, per-frame sky gradient, and DOM actor
  style writes in `syncActors()`.** All bounded by small collections (≤20 robots,
  tens of projectiles). Rewriting them into pooled/in-place structures would be
  a large, risky diff for no measurable gain at this scale.
- **No touch controls for gameplay.** The game states "Desktop keyboard
  required"; hot-seat keyboard play is its design. Menus and weapon choice
  already work with the mouse/touch via pointer events.
- **No difficulty levels / no changes to weapons, robots, terrain, music,
  names or text.** Content and balance are the author's; the revision only
  fixes how the game runs and what it communicates.
- **Existing tuning constants were not moved** into `CONFIG` (see §A) — they
  were already named and grouped with their systems.

---

## D. Test results

The whole diff is **108 changed lines in a 4,087-line file** — the game is
untouched except for the fixes above.

**Method note (honest):** a live browser play-test was attempted first, but the
sandbox's browser cannot be driven directly and downloading a separate browser
was declined, so verification was done with (1) a syntax check, (2) a pure-Node
behaviour harness that loads the *real* game script with stubbed DOM/canvas/audio
and drives it through its own event handlers and animation-frame timestamps, and
(3) a local static HTTP server for asset checks. Rendering/pixels and real sound
could not be verified this way — see "not verified" below.

1. **Syntax:** the game's script extracted and checked with `node --check` — ✅ valid.
2. **Load:** script loads with **zero errors**; initial state `presentation`. ✅
3. **Delta-time loop (fix 6), measured from the game's own frame counter:**
   - 60 Hz timestamps for 1 s → **59 simulation steps** ✅ (≈60; the first frame only initialises the clock)
   - 120 Hz timestamps for 1 s → **60 simulation steps** ✅ (the original loop would have run 120 — double speed)
   - One 5-second timestamp jump (background tab) → **exactly 5 steps** ✅ (clamp 0.25 s + cap 5: no teleport, no spiral)
4. **State machine:** presentation → (key press) → `menu` → (Start battle) →
   `play`. ✅
5. **Pause / resume:** `P` → paused ✅; `Esc` → resumed ✅ (new binding works);
   `Esc` again → paused ✅.
6. **Restart:** clicking RESTART in the pause menu (through the game's real
   canvas pointer handler) → back in `play`, unpaused, frames advancing. ✅
7. **Assets over HTTP** (local static server, revised folder): `index.html`,
   all four MP3s and the title PNG return **200**, including
   `Triomphe des Alli%C3%A9s.mp3` (4,132,736 bytes — the full file). ✅
   The same server confirms the audit finding: the old mangled filename
   (`…Alli#U00e9s.mp3`) returns **404**.
8. **Smaller-viewport overflow: NOT verified live** (needs a real browser).
   The CSS was not changed in this area — the stage already scales with
   `width:min(100vw, calc((100vh - 36px)*2))` and the 2:1 aspect is preserved —
   so behaviour is identical to the original, which was built for this. Flagged
   here rather than claimed.
9. **Not verified live:** actual rendering/pixel output, audible sound/music
   playback, the game-over screen's new history line (fix 3) rendering, and a
   full match played to completion. The history *logic* is exercised only as
   far as `recordMatchResult()` being called from `endGameCheck()`; the line
   itself is drawn with the same canvas text calls as the rest of that screen.
   These are the first things to eyeball when the game is next opened in a
   real browser.
