# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**PRS-Game** — a Rock · Paper · Scissors (+ Lizard · Spock) game built as a single self-contained HTML file (`prs-game.html`). No build step, no dependencies, no external assets. Everything is inline HTML/CSS/JS. The player throws a move against a fair, random computer opponent. Sibling project to **Word Strike** (the typing game) and reuses its dependency-free conventions: procedural Web Audio sound, `localStorage` persistence, mobile support, and English/Arabic (RTL) bilingual UI.

The active theme is **"Solar Flare"** — amber/burnt-orange on near-black. All colors are CSS custom properties in `:root`; the theme can be swapped by replacing those vars only. Previous themes: Arcade Duel (indigo/violet), Crimson Night (red/rose).

## Running locally

Python's built-in HTTP server (Node/npm is not available in this environment — Python is the only runtime):

```
python -m http.server 3939
```

Then open `http://localhost:3939/prs-game.html`. The `.claude/launch.json` is configured for the `preview_start` tool under the name `prs-game`.

## Deployment

`vercel.json` rewrites `/` → `/prs-game.html` so the root URL serves the game. GitHub repo: `git@github.com:mo-io/prs-game.git` (private, SSH auth, account `mo-io`). Vercel is connected to the repo via the dashboard and auto-redeploys on every push to `master`. No Vercel CLI is used or needed.

## Game overview

Two fighter cards face off ("YOU" vs "CPU"). The player throws a move via buttons, keyboard, or number keys. Both cards play a "Rock, Paper, Scissors, Shoot!" shake animation, then reveal the chosen glyphs and resolve the round. A result banner shows **You win! / CPU wins! / Draw**.

**Variant** (chosen on start overlay):
- **Classic** — Rock ✊ / Paper ✋ / Scissors ✌️ (keys R/P/S or 1/2/3)
- **Lizard-Spock** — adds Lizard 🦎 / Spock 🖖 (keys L/K); 10 win relationships

**Match formats** (chosen on start overlay):
- **Best of N** (3 / 5 / 7) — first to `ceil(N/2)` round wins ends the match → game-over overlay.
- **Endless** — never auto-ends; tracks current win **streak** and persists the **best streak**.

**HUD:** YOU score · center mode badge (Best-of target, or live streak) + best-streak chip (★) · CPU score.

## Architecture

`prs-game.html` is self-contained. Key pieces:

### Rules as data (extensibility)

Win logic is pure data. The full 5-move set is already in the file:

```js
const MOVES_CLASSIC = ['rock','paper','scissors'];
const MOVES_LS      = ['rock','paper','scissors','lizard','spock'];
const BEATS = {
  rock:    ['scissors','lizard'],
  paper:   ['rock','spock'],
  scissors:['paper','lizard'],
  lizard:  ['spock','paper'],
  spock:   ['scissors','rock'],
};
const GLYPH = { rock:'✊', paper:'✋', scissors:'✌️', lizard:'🦎', spock:'🖖' };
const beats = (a,b) => BEATS[a].includes(b);
let MOVES = MOVES_CLASSIC;   // reassigned by updateVariant()
```

`cpuPick()` returns a uniform-random move from the active `MOVES` array — fair, no streak manipulation. `MOVES` is `let` (not `const`) so `updateVariant()` can swap it at game start.

### State machine

Two booleans and a lock govern everything: `state.running`, `state.paused`, `state.locked`.

| State | `running` | `paused` | `locked` |
|-------|-----------|----------|---------|
| Idle / game over | `false` | `false` | `false` |
| Playing (between rounds) | `true` | `false` | `false` |
| Playing (reveal animation) | `true` | `false` | `true` |
| Paused | `true` | `true` | `false` |

Full `state` object:
```js
{ language, running, paused, locked, mode, bestOf,
  youScore, cpuScore, round, streak, bestStreak,
  variant,   // 'classic' | 'ls'
  audioCtx }
```

Module-level: `overlayMode` (`'start'` | `'gameover'`) tells `applyOverlayStrings()` which text set to render. `lastResult` (`'win'`/`'lose'`) colors the game-over title.

### Lifecycle functions

| Function | What it does |
|----------|-------------|
| `startGame()` | Inits audio, calls `updateVariant()`, resets scores/round/streak, hides overlay, shows `#game-btns`, enables controls |
| `updateVariant()` | Sets `MOVES` to `MOVES_CLASSIC` or `MOVES_LS`; toggles `.ls-active` on `#controls` to show/hide LS buttons |
| `pauseGame()` | Only fires when `!locked`; sets `paused`, disables controls, shows `#pause-overlay`, flips Pause btn label to Resume |
| `resumeGame()` | Clears `paused`, hides `#pause-overlay`, re-enables controls |
| `playRound(move)` | Locks input, picks CPU move, runs shake→reveal animation, calls `resolveRound()` |
| `resolveRound(result)` | Updates scores/streak/banner/sound; checks best-of match end; unlocks controls (unless paused) |
| `endMatch(result)` | Stops match, hides `#game-btns`, shows game-over overlay |
| `resetGame()` | Full teardown → start overlay; clears `running`, `paused`, `locked`, hides `#game-btns` |
| `syncGameBtns()` | Updates Pause/Resume label+icon, Reset label, pause overlay title — call after any state or language change |
| `updateHUD()` | Refreshes scores, mode badge, best-streak chip — call after every score/mode change |

**Rule:** After every state transition, call `syncGameBtns()` and/or `updateHUD()` as needed. Missing a call leaves the UI stale.

### Round flow / animation
No persistent game loop. A round is driven by `setTimeout`/`setInterval`: both cards show ✊ with the `shake` CSS class while `sndTick()` fires 3 times (~760ms total), then real glyphs are set with a `pop` animation and the outcome resolves. CPU card is mirrored (`scaleX(-1)`) so fists face each other — its shake/pop keyframes have `-cpu` variants to preserve the mirror.

`state.locked` is set at the start of `playRound()` and cleared in `resolveRound()`'s delayed callback. Pause (`Esc`) is ignored while `locked` is true; the current animation completes first.

### Lizard-Spock variant UI

The start overlay has a `#variant-seg` segmented control (Classic / Lizard-Spock). Switching it:
1. Sets `state.variant`
2. Calls `updateVariant()` — swaps `MOVES`, toggles `#controls.ls-active`
3. Calls `applyOverlayStrings()` — swaps instructions text for the LS ruleset

Lizard and Spock buttons have the class `.ls-only`. CSS rule:
```css
.ls-only { display: none; }
#controls.ls-active .ls-only { display: flex; }
#controls.ls-active .move-btn { max-width: 130px; }
```

Mobile (≤520px) in LS mode: `#controls.ls-active` wraps to a 3+2 grid (`flex-wrap:wrap`; each button `width:calc(33.33% - 10px); flex:0 0 auto`). `.mv-name` labels are hidden in LS+mobile to save space — glyph alone is sufficient.

### Audio
Procedural via Web Audio (`state.audioCtx`), lazily created in `initAudio()` on first interaction (autoplay policy). `playTone(freq, type, dur, vol, delay)` — same helper as Word Strike. Events: `sndTick` (shake), `sndWin` (ascending arpeggio), `sndLose` (saw thud), `sndDraw` (square blip), `sndMatchWin` (chord), `sndMatchLose` (descending saw).

### Persistence
`localStorage` key **`prs_best`** — best endless win streak only. `loadBest()` on init, `saveBest()` when a new best is reached. Best-of match results are session-scoped and not persisted.

### Language support (en / ar)
- `STRINGS.en` / `STRINGS.ar` — all visible text. Some entries are functions: `bestOf(n)`, `best(n)`. LS-specific keys: `variantLabel`, `classic`, `ls`, `lizard`, `spock`, `instructionsLS`.
- `applyOverlayStrings()` — applies active language to the overlay; toggles `#overlay.rtl`; in start mode also syncs both `#mode-seg` and `#variant-seg` button labels; switches instructions to `instructionsLS` when `state.variant === 'ls'`.
- `updateLangButton()` — syncs `.lang-btn` active state, sets `<html lang>`, calls `applyOverlayStrings()` + `syncGameBtns()`.
- When adding a new string: add to **both** `STRINGS.en` and `STRINGS.ar` in parallel, then reference via `STRINGS[state.language].myKey`.

### Visual / CSS
- All colors are CSS custom properties on `:root` (`--accent`, `--accent-2`, `--win`, `--lose`, `--draw`, `--ink`, `--muted`, `--card`, `--card-hi`, `--bg-0/1/2`, `--line`, `--shadow`) — retheme by replacing these 14 vars only.
- Responsive: `@media (max-width:520px)` compacts HUD, hides `.gb-label`/`.gb-key` on game buttons, hides `.mv-key` on move buttons, and applies the LS 3+2 grid; `@media (max-height:560px)` tightens the arena.
- `viewport-fit=cover` + `env(safe-area-inset-*)` padding for notched phones.
- **Do not use `background-attachment: fixed`** — causes iOS Safari repaint jank and stalls the headless preview tool.
- `.hidden` class = `display:none` (used on `#overlay`, `#pause-overlay`, `#game-btns`).

### z-index stack
| Layer | z-index |
|-------|---------|
| Game elements (HUD, arena, controls) | 0–10 |
| `#pause-overlay` | 15 |
| `#overlay` (start / game-over) | 20 |

## Keyboard shortcuts
| Key | When | Action |
|-----|------|--------|
| `R` / `P` / `S` (or `1`/`2`/`3`) | Playing — Classic or LS | Throw Rock / Paper / Scissors |
| `L` / `K` | Playing — LS mode only | Throw Lizard / Spock |
| `Esc` | Playing | Pause |
| `Esc` / `Enter` / `Space` | Paused | Resume |
| `Enter` | Start or game-over overlay | Start / Play Again |

L and K keys are in `KEYMAP` but gated: `MOVES.includes(KEYMAP[k])` is checked before firing, so they're silently ignored in Classic mode.

---

## Handoff Notes

> Read this section when picking up the project after a gap. It captures the current state, reasoning behind non-obvious choices, established conventions, and concrete next steps.

### Current project state (as of 2026-06-30)

The game is **complete and live**. `prs-game.html` is ~830 lines of self-contained HTML/CSS/JS. Deployment: GitHub `mo-io/prs-game` (private) → Vercel auto-deploy on push to `master`.

**What is fully working:**
- Classic RPS and **Lizard-Spock** (5-move) variant — switchable on the start overlay via VARIANT segmented control
- Best-of-3/5/7 and Endless match modes
- Pause / Resume / Reset in-game (Pause gated by `state.locked` — no mid-animation interrupts)
- Shake→Reveal→Resolve round animation (~760ms); both cards show ✊ during shake regardless of move
- Procedural Web Audio (6 distinct sound events)
- Full English/Arabic bilingual UI with RTL support, including all LS move names and rule descriptions
- Mobile responsive (≥64px tap targets in LS mode, 3+2 grid layout, safe-area padding, compact HUD at ≤520px)
- `prs_best` localStorage streak persistence (Endless mode only)
- `#game-btns` (Pause + Reset) visible only during active play
- `#pause-overlay` with Resume + Reset, semi-transparent
- **Solar Flare** color theme (amber/burnt-orange) — 14 CSS vars in `:root`, trivially rethemeable

### Key decisions and why

**Single self-contained HTML file**
Same pattern as sibling project Word Strike. Zero dependencies, no build tooling, Python HTTP server only. Never split into multiple files — the entire game must remain in `prs-game.html`.

**`MOVES` is `let`, not `const`**
`updateVariant()` needs to swap `MOVES` between `MOVES_CLASSIC` and `MOVES_LS` at game start. Making it `let` allows reassignment; closures over `MOVES` (like `cpuPick`) correctly see the updated value. Do not hardcode array references to `MOVES_CLASSIC` or `MOVES_LS` in game logic — always use `MOVES`.

**Win rules as data, not logic**
`BEATS`, `GLYPH`, and `beats()` are the only things determining outcomes. No `if (move === 'lizard')` branches anywhere in the game logic. Adding a future move (e.g. a 7-move variant) requires only extending the data objects and adding HTML buttons — no logic changes.

**`.ls-only` visibility pattern**
Lizard/Spock buttons exist in the DOM at all times; they're hidden via `.ls-only { display:none }` and revealed by toggling `.ls-active` on `#controls`. This avoids dynamically creating/destroying DOM elements and keeps `moveBtns` (built at page load via `querySelectorAll('.move-btn')`) stable and complete.

**Keyboard input gated by `MOVES.includes()`**
`l` and `k` are in `KEYMAP` unconditionally, but the handler checks `MOVES.includes(KEYMAP[k])` before firing. This means LS keys are silently ignored in Classic mode — no special casing needed when adding future moves.

**Solar Flare palette (was Arcade Duel)**
The user chose Solar Flare (amber/orange) over the original indigo/violet Arcade Duel theme in this session. Two other palettes were shown (Neon Reef, Deep Space) and not selected. All 14 `:root` vars drive the full visual — changing theme means only changing those vars plus the comment on line 9.

**Endless-only streak persistence**
Only Endless mode writes to `prs_best`. Best-of-N results are session-scoped. If you add Best-of persistence, use a separate `localStorage` key (e.g. `prs_bestof_record`).

**Pause gated by `state.locked`**
`pauseGame()` returns early if `state.locked` is true (during the ~760ms reveal animation). The round is short enough that waiting is acceptable UX — avoids complexity of cancelling in-flight `setTimeout` timers.

**No `background-attachment: fixed`**
Causes iOS Safari repaint jank and stalls the headless preview screenshot tool. Keep backgrounds as simple gradient combos without this property.

### Coding patterns and conventions

**Adding a new UI string:**
1. Add to `STRINGS.en` and `STRINGS.ar` simultaneously (never one without the other)
2. Reference as `STRINGS[state.language].myKey` — never hardcode the string inline
3. Call `applyOverlayStrings()` or `syncGameBtns()` (whichever is relevant) after language changes

**Adding a new lifecycle state transition:**
1. Update `state.running`, `state.paused`, `state.locked` as appropriate
2. Show/hide `#overlay`, `#pause-overlay`, `#game-btns` with `.classList.add/remove('hidden')`
3. Add/remove `disabled` class from `#controls`
4. Call `syncGameBtns()` + `updateHUD()` at the end

**Adding a new sound:**
Use `playTone(freq, type, dur, vol, delay)` — do not create oscillator setup from scratch. Chain multiple calls with staggered `delay` values for arpeggios. Wrap in `const sndXxx = () => ...`.

**Adding a new CSS component:**
Define all colors via existing `--var` references from `:root`. Never hardcode hex colors in component rules. Add a `@media (max-width:520px)` override for anything that needs to shrink or hide on mobile.

**DOM shorthand:** `const $ = id => document.getElementById(id)`. Use `document.querySelector` / `querySelectorAll` for class/attribute selectors.

**`.hidden` pattern:** Toggle with `.classList.add/remove('hidden')`. Never toggle `style.display` directly on elements that use this pattern.

**Variant-aware code:** When writing logic that touches moves or move counts, always use `MOVES` (the active array), never hardcode `['rock','paper','scissors']`.

### Specific next steps (implementation-ready)

#### 1. Mute / sound toggle (low effort)

**Step 1** — add `muted:false` to `state`. Load from `localStorage` key `prs_muted` on init.

**Step 2** — gate `playTone()`:
```js
function playTone(freq, type='sine', dur=0.06, vol=0.12, delay=0){
  if (!state.audioCtx || state.muted) return;
  // ... rest unchanged
}
```

**Step 3** — add a mute button to `#game-btns`:
```html
<button class="game-btn" id="btn-mute"><span class="gb-icon">🔊</span><span class="gb-label">Mute</span></button>
```
Toggle handler: `state.muted = !state.muted; localStorage.setItem('prs_muted', state.muted); syncGameBtns();`

**Step 4** — update `syncGameBtns()` to set `#btn-mute` icon to `🔇` when `state.muted`.

**Step 5** — add to `STRINGS.en/ar`: `mute:'Mute'`/`'كتم الصوت'`, `unmute:'Unmute'`/`'تشغيل الصوت'`.

---

#### 2. Streak milestone celebrations (low effort)

In `resolveRound()`, after the `state.streak++` line, add:
```js
if (state.mode === 'endless' && state.streak > 0 && [5,10,20,50].includes(state.streak)){
  showMilestone(state.streak);
}
```

`showMilestone(n)` — create a `#milestone-banner` div positioned above `#banner`, briefly show it with a CSS class, then clear it after ~1.5s. Example content: `` `🔥 ${n} streak!` `` / Arabic: `` `🔥 ${n} انتصارات متتالية!` ``. Add both strings to `STRINGS.en/ar`.

---

#### 3. Match-point alert (low effort)

In `resolveRound()`, after score update and before the match-end check:
```js
if (state.mode === 'bestof'){
  const target = Math.ceil(state.bestOf / 2);
  youCard.classList.toggle('match-point', state.youScore === target - 1);
  cpuCard.classList.toggle('match-point', state.cpuScore === target - 1);
}
```

Add CSS:
```css
.glyph-card.match-point {
  border-color: var(--draw);
  animation: pulse-border 0.8s ease-in-out infinite;
}
@keyframes pulse-border {
  0%,100%{ box-shadow: 0 0 0 2px rgba(255,215,0,.3), var(--shadow); }
  50%    { box-shadow: 0 0 0 6px rgba(255,215,0,.6), var(--shadow); }
}
```

Clear `.match-point` at the top of `resetCards()`.

---

#### 4. Predictive AI / difficulty (moderate effort)

**Step 1** — add to `state`: `aiMode:'random'`, `moveHistory:[]`. Reset `moveHistory` in `startGame()`.

**Step 2** — smart pick:
```js
function cpuPickSmart(){
  if (state.moveHistory.length < 5) return cpuPick();
  const counts = Object.fromEntries(MOVES.map(m=>[m,0]));
  state.moveHistory.slice(-10).forEach(m=>counts[m]++);
  const likely = Object.keys(counts).reduce((a,b)=>counts[a]>counts[b]?a:b);
  return MOVES.find(m=>BEATS[m].includes(likely)) ?? cpuPick();
}
```

**Step 3** — in `playRound()`, push move to history: `state.moveHistory.push(move);` (before `cpuPick`).

**Step 4** — in `playRound()`: `const cpu = state.aiMode==='smart' ? cpuPickSmart() : cpuPick();`

**Step 5** — add DIFFICULTY segmented control to start overlay (Easy/Hard). `state.aiMode = b.dataset.ai`. Add `STRINGS.en/ar` entries: `difficulty`, `easy`, `hard`.

Note: `cpuPickSmart` already uses `MOVES` so it works for both Classic and LS variants without changes.

---

#### 5. Game-over visual impact (low effort)

The current game-over overlay shows raw scores. A big "YOU WIN" / "CPU WINS" heading would be more satisfying. In `applyOverlayStrings()`, in the `gameover` branch, set `#ov-title` to a larger stylized text and color it with `--win` or `--lose`. Add a subtitle line showing the score. This is purely a string/style change — no logic needed.

### Files that need attention next

| File | Why |
|------|-----|
| `prs-game.html` | The only code file — all features, styles, and logic live here |
| `MEMORY.md` | Update "Current State" and "Build History" after each session |
| `CLAUDE.md` | Update keyboard shortcuts, state object, and lifecycle tables when they change |

`MEMORY.md` and `CLAUDE.md` are not auto-updated. Amend both manually before committing after shipping a feature.
