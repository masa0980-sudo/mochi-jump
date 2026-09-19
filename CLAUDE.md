# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A short browser 2D platformer: a bouncy mochi character crosses hand-built stages (pits, spikes,
moving platforms, collectible "きなこ" dots) to reach each stage's goal (a mochi-pounding usu/mortar).
The entire app — HTML, CSS, JS, Canvas rendering — lives in a single `index.html` file with no build
step, no bundler, and no dependencies, following the same philosophy as the sibling `Rhythm_game`,
`tennis-game`, and `neon-void` repos. Published as-is to GitHub Pages.

## Commands

There is no package.json, build step, linter, or test framework — this is intentional; keep it
that way rather than introducing tooling.

- **Run locally**: serve the repo root and open `index.html`, e.g. `python3 -m http.server 8000`.
- **Verify UI/gameplay changes**: use Playwright to drive a real browser against the local server
  (there is no automated test suite). The browser is pre-installed — launch with
  `executablePath: "/opt/pw-browsers/chromium"` and do **not** run `playwright install`.
- **Deploy**: push to `main`. GitHub Actions (`.github/workflows/deploy-pages.yml`) rebuilds and
  republishes GitHub Pages automatically — there is no separate deploy command.

## Architecture

Everything is one `<script>` inside `index.html`, organized as numbered, comment-delimited
modules, in dependency order:

1. **Sfx** — generates sound via the Web Audio API (no audio files).
2. **PlayCounts** — Firestore-backed play counter, see below.
3. **Input** — unifies keyboard (arrows/space/wasd) and touch (on-screen buttons) input.
4. **Level / LEVELS** — `LEVELS` is an array of stage definitions (`id`, `name`, `desc`, plus the
   raw per-stage data: `platforms` (ground-like rects, only their *top* is solid), `movers`
   (platforms that oscillate between `x0`/`x1` at `vx` px/s), `spikes` (instant-death rects),
   `pits` (visual + fall-detection ranges — not real physics, just "if the player's y exceeds
   `GROUND_Y + 140` here, they've fallen"), `coins` ("きなこ" collectibles, optional bonus),
   `goalX`/`levelW`). `loadLevel(idx)` copies `LEVELS[idx]` into the *working* module-scoped
   `let platforms, movers, spikes, pits, coins, GOAL_X, LEVEL_W` (cloning each object — including
   resetting every mover's `dir` to `1` and every coin's `taken` to `false` — so replaying a stage
   or switching stages never leaks state from a previous run). All game code (`update()`,
   `render()`, `respawnPointFor()`, etc.) reads these working `let` bindings, never `LEVELS`
   directly — `LEVELS[idx]` is treated as read-only template data.
4b. **Progress** — tiny `localStorage` wrapper (`mochi-jump:progress`, `{cleared: [id, ...]}`)
   tracking which stage ids have been cleared at least once. `paintStageList()` uses this to lock
   stage *N+1* until stage *N*'s id is in `cleared`.
5. **Game** — physics (`update()`), collision (`aabbTop` + a "was above last frame, is at/below
   now" landing check — deliberately simple, only ever lands on top of a platform, never
   side/bottom collision), camera (`updateCamera`, lerps toward the player), rendering
   (`render()`, all Canvas 2D draw calls — the mochi and every obstacle are still drawn
   procedurally, no sprite art to keep in sync). `drawBg()` only paints a few parallax star dots
   and never fills a background rect, so `canvas#game`'s CSS `background` (set per-stage in
   `loadLevel()` from `LEVELS[idx].bg`) shows through underneath everything the canvas draws —
   see "Art" below.
6. **Screen switching**: `SCREENS = ["title","select","play","result"]` + `show(id)`. Any new
   screen must be added to `SCREENS`. `select` is the stage-list screen; `startGame(levelIndex)`
   is the one place that calls `loadLevel()` — always go through it rather than mutating the
   working level state directly.

### Coordinate system — the thing most likely to bite you

The logical canvas resolution is `VIEW_W = 640, VIEW_H = 420` (see `fitCanvas()`, which just
scales this via CSS width/height to fit the device — game logic never touches device pixels).
`GROUND_Y = 320` is the *top* of ground-level platforms, shared by every stage in `LEVELS`.
**Any level geometry you add must stay within roughly y ∈ [0, 420]** or it silently renders
off-canvas (this bit the very first version: an earlier `GROUND_Y = 460` with `VIEW_H = 400` put
the entire level below the visible area with no error — nothing crashes, it just draws nothing you
can see. If a change makes a stage look empty, check this before anything else).

### Jump physics — known reach, use it when designing a stage

Before adding/editing a pit or platform height, know the actual limits (simulated frame-by-frame
against the real constants `GRAVITY=0.62, MOVE_ACC=0.55, MOVE_MAX=4.4, JUMP_V=-11.2,
JUMP_HOLD_ACC=-0.52, JUMP_HOLD_MAX_T=14`, landing back at launch height):

| jump-button hold | horizontal reach (cold start) | horizontal reach (full run-up) |
|---|---|---|
| tap (1 frame) | ~147px | ~163px |
| ~half hold (7 frames) | ~191px | — |
| full hold (14 frames) | ~231px | ~246px |

Max vertical rise on a full-hold jump is ~219px above the launch height. **Every non-mover pit
across every stage must stay at or under ~231px** (the conservative cold-start bound — don't rely
on the player having a run-up) — `stage3`'s `700→920` pit (220px) is deliberately right at that
edge as the "hardest stage" signature jump; don't add another gap this tight without a good reason.
Wider gaps need a `movers` entry to bridge them instead (see `stage2`'s `2150→2450` and `stage3`'s
two mover-bridged gaps for the pattern: the mover's oscillation range should comfortably overlap
both the departure platform's far edge and the arrival platform's near edge).

If a stage "looks empty" in testing or a jump that should clearly work doesn't, re-derive this
table with a quick Node simulation rather than guessing — it's cheap and several real bugs during
this stage's development turned out to be test-script issues (wrong run-up distance landing in an
adjacent gap, chained repeated max-jumps overshooting a platform's collision at high fall speed)
rather than actual level problems, precisely because the physics were verified against this table
first.

### Death / respawn model

There's no lives/game-over state — falling in a pit or touching a spike sets `player.dead = true`,
plays a sound, and after `deadT` frames respawns the player at `lastRespawnX` (the rightmost
ground platform's edge the player has actually stood on, tracked via `respawnPointFor()` — updated
only when `player.onGround && !player.dead`, so a mid-air teleport-style test that never lands
won't move the checkpoint). This is deliberate: the title screen promises "落ちても大丈夫" (falling
is fine), so keep retries cheap and frequent rather than punishing.

### PlayCounts (Firestore)

Same no-SDK, `fetch()`-only approach as the sibling repos — no Firebase SDK is loaded, just raw
Firestore REST calls. Project ID `rythm-game-mo`; the API key is meant to be public in client
code. `increment("mochi-jump")` is called once, in `startGame()`, which is the single function
both the title screen's "はじめる" and the result screen's "もういちど" call — so both a fresh
start and a retry count as a play, with no duplicated call site. It tries the atomic `+1` field
transform first (the steady-state case, one request); if that fails because the doc doesn't exist
yet (this app's very first play, ever), it falls back to creating the doc with `count: 1` — the
security rule on the shared project allows creating a doc *only* when it's exactly `{count: 1}`,
so a brand-new gameId self-registers with zero manual Firebase-console setup. `fetchCount()` is a
plain read-only GET of the single `playCounts/mochi-jump` document, called once on load to fill in
the title screen's "これまでに ◯ 回プレイされています" line — non-blocking, and left blank forever
if the fetch fails or the doc doesn't exist yet (no loading spinner, no retry).

**`increment()` is a no-op when served from `localhost` / `127.0.0.1` / `file:`** (2026-09-19).
The Firestore target is the hardcoded *production* project, so every Playwright check that called
`startGame()` during stage development (see the testing patterns above — "always call
`startGame(idx)` immediately before each independent test") sent a real `+1`. The public counter
had reached ~25 plays before the game was linked anywhere, which is how this was noticed.
`fetchCount()` is deliberately left unguarded so the title screen still shows the real number
during local testing. If you ever need to test the write path itself, temporarily serve from a
non-loopback hostname rather than removing the guard.

This Firestore project is intentionally **shared across several separate public games**
(`Rhythm_game`, `tennis-game`, `neon-void`, `typing_quotes`, etc.) — each game's own
`playCounts/{gameId}` document lives in the same `playCounts` collection, keyed by an id unique to
that game. This repo uses `"mochi-jump"`. If you ever add a second mode/stage that should be
counted separately, give it its own id (check it doesn't collide with another game's) rather than
reusing `"mochi-jump"`.

### `window.__mochi` — debug/test hook

Exposes `player()`, `teleport(x,y)`, `coins()`, `coinCount()`, `goalX()`, `levelW()`, `groundY`,
`spikes()`, `pits()`, `movers()`, `isEnded()`, `currentLevel()`, `levels` (the raw `LEVELS` array),
`startGame(idx)`, `clearProgress()`, and `step(n, {left,right,jumpHeld,jumpPress})` — unconditionally
(not gated behind a `?debug=1` flag, unlike `Rhythm_game`'s `window.__rhythmDebug`), same idea as
`window.__tennis` in `tennis-game`. **Everything that depends on the current stage is a function,
not a plain value** — `platforms`/`movers`/`spikes`/`pits`/`coins`/`GOAL_X`/`LEVEL_W` are all
reassigned by `loadLevel()` on every `startGame()` call, so a plain captured value would go stale
the moment a second stage loads.

Two testing patterns worth knowing before you next touch level geometry:
- `teleport()` + realtime input is fine for one-off checks, but **always call `startGame(idx)`
  immediately before each independent test** — `player.dead` and other run state carry over
  between checks otherwise (an earlier death state silently no-ops all later `update()` calls
  until the respawn timer clears, making an unrelated later test look like it failed).
- `step(n, input)` advances `update()` exactly `n` times at a fixed 16.6667ms/frame, bypassing
  real keyboard timing entirely — use it for anything where exact frame counts matter (verifying a
  specific pit's width is actually crossable with a specific jump-hold duration, per the reach
  table above). Realtime `keyboard.down()`/`waitForTimeout()` is fine for a casual look at a
  stage, but don't trust its precise pass/fail for a tight jump — wall-clock jitter changes the
  effective hold duration by a frame or two, which is exactly the margin some of these jumps live
  in.

## Scope

Ships with three stages (`stage1`/`stage2`/`stage3` in `LEVELS`, unlocked in order via `Progress`).
Still no lives system and no leaderboard — falling/dying only costs a respawn, never a life or a
run-ending game-over, and there's no cross-stage scoring beyond the per-run "きなこ" count and
clear time shown on the result screen. Adding a fourth stage means appending to `LEVELS` (respecting
the reach table above) — `paintStageList()`, `Progress`, and `startGame()` all already generalize
over `LEVELS.length` and need no changes for additional stages.

## Art

`art/keyart.png` (title screen, set as `#title`'s CSS `background`, dimmed with a `#title::before`
scrim gradient for text legibility) and `art/bg_stage{1,2,3}.png` (one per `LEVELS` entry, via
`bg:` field, applied to `canvas#game` in `loadLevel()`) are backdrop illustrations only — no
sprites, no gameplay hitboxes derive from them. All are picture-book (ehon) style pastel
wagashi-world illustrations generated with Gemini's web app (image API has no free tier; see the
sibling repos' CLAUDE.md "Gemini でシーン画像を無料で作る" for the free route). Missing/failed-to-load
art degrades silently: CSS background-image that 404s just doesn't paint, leaving the plain
gradient (`#title` falls back to `body`'s own radial-gradient since it sets no background of its
own without the image; `canvas#game`'s inline `style.background` always includes the original navy
gradient as a second layer) — no JS image-loading/fallback logic needed, unlike the sprite
`poseFile()`/`missingArt` pattern in `tennis-game`.

**`bg_stage{1,2,3}.png` are deliberately calm/sparse** (2026-09-19 revision): the first version
used `keyart.png` as a reference image so the busy hero-illustration composition (dense wagashi
clusters, the mochi character drawn into the scene) carried straight into the stage backgrounds,
and it read as cluttered *behind actual gameplay* — competing with the platforms/spikes/coins the
canvas draws on top. Revised prompts keep only a thin band of scenery hugging the very bottom edge
(low hill silhouette, one or two sparse trees) and leave the middle/lower two-thirds of the frame
almost empty sky, explicitly excluding the character and any food/prop clutter, so gameplay stays
readable. **The original busy illustrations weren't thrown away** — they were repurposed as
`art/clear_stage{1,2,3}.png`, shown via `#resultClearArt` in `showResult()` only when `cleared` is
true (via each `LEVELS` entry's `clearArt` field) — a "richer scene as the reward for finishing"
reads well precisely because it's *not* what you were staring at while playing. Same graceful
degradation as the other art: `img.onerror` just re-hides the element.

## BGM

`Sfx` also owns a single always-on pop BGM (`startBgm()`/`stopBgm()`, called from `startGame()`,
`showResult()`, pause/resume/quit, `visibilitychange`, and `pagehide`) — same look-ahead scheduler
architecture as `tennis-game`/`neon-void` (`SCHEDULE_AHEAD=0.6s`, catch-up-by-resync instead of
bursting stale notes, a shared noise buffer/filters, `onended`-triggered `disconnect()`). One
4-bar loop, BPM 148, the classic pop progression C→G→Am→F voiced as 7th chords (`CHORDS[].tones`
has 4 entries, not 3) so the pad layer alone has some harmonic thickness; a second, barely-detuned
(`*1.004`) sine layer doubles every pad note for a light chorus effect — that pairing is what makes
it not read as "monotonous" the way a single square-wave lead over bare triads would. Measured
~43 nodes/s, comfortably under neon-void's documented worst case (219–306/s), so no per-stage
theme switching was needed to keep it light — see `tennis-game`'s CLAUDE.md if a future request
wants stage-by-stage BGM variety, that repo's `bgmThemeFor()` pattern is the template to copy.
