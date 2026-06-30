# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**PRS-Game** — a Rock · Paper · Scissors game built as a single self-contained HTML file (`prs-game.html`). No build step, no dependencies, no external assets. Everything is inline HTML/CSS/JS. The player throws a move against a fair, random computer opponent. Sibling project to **Word Strike** (the typing game) and reuses its dependency-free conventions: procedural Web Audio sound, `localStorage` persistence, mobile support, and English/Arabic (RTL) bilingual UI.

Visual identity is the **"Arcade Duel"** theme — an indigo/violet gradient, bold rounded cards, and large emoji gesture glyphs — deliberately distinct from Word Strike's neon cyberpunk look.

## Running locally

Python's built-in HTTP server (Node/npm is not available in this environment — Python is the only runtime):

```
python -m http.server 3939
```

Then open `http://localhost:3939/prs-game.html`. The `.claude/launch.json` is configured for the `preview_start` tool under the name `prs-game`.

## Deployment

`vercel.json` rewrites `/` → `/prs-game.html` so the root URL serves the game. GitHub repo: `git@github.com:mo-io/prs-game.git` (private, SSH auth, account `mo-io`). Vercel is connected to the repo via the dashboard and auto-redeploys on every push to `master`. No Vercel CLI is used or needed.

## Game overview

Two fighter cards face off ("YOU" vs "CPU"). The player throws **Rock ✊ / Paper ✋ / Scissors ✌️** via the three move buttons, the `R`/`P`/`S` keys, or `1`/`2`/`3`. Both cards play a "Rock, Paper, Scissors, Shoot!" shake animation, then reveal the chosen glyphs and resolve the round. A result banner shows **You win! / CPU wins! / Draw**.

**Match formats** (chosen on the start overlay):
- **Best of N** (3 / 5 / 7) — first to `ceil(N/2)` round wins ends the match → game-over overlay.
- **Endless** — never auto-ends; tracks current win **streak** and persists the **best streak**.

**HUD:** YOU score · center mode badge (Best-of target, or live streak) + best-streak chip (★) · CPU score.

## Architecture

`prs-game.html` is self-contained. Key pieces:

### Rules as data (extensibility)
The win logic is pure data so the 5-move **Lizard-Spock** variant can be added with no structural change:
```js
const MOVES = ['rock','paper','scissors'];
const BEATS = { rock:['scissors'], paper:['rock'], scissors:['paper'] };
const GLYPH = { rock:'✊', paper:'✋', scissors:'✌️' };
const beats = (a,b) => BEATS[a].includes(b);
```
To add Lizard-Spock: extend `MOVES`, `BEATS`, `GLYPH`, add two move buttons + their `STRINGS` labels. No logic changes.

`cpuPick()` returns a uniform-random move — fair, no streak manipulation.

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
  youScore, cpuScore, round, streak, bestStreak, audioCtx }
```

Module-level: `overlayMode` (`'start'` | `'gameover'`) tells `applyOverlayStrings()` which text set to render. `lastResult` (`'win'`/`'lose'`) colors the game-over title.

### Lifecycle functions

| Function | What it does |
|----------|-------------|
| `startGame()` | Inits audio, resets scores/round/streak, hides overlay, shows `#game-btns`, enables controls |
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

### Audio
Procedural via Web Audio (`state.audioCtx`), lazily created in `initAudio()` on first interaction (autoplay policy). `playTone(freq, type, dur, vol, delay)` — same helper as Word Strike. Events: `sndTick` (shake), `sndWin` (ascending arpeggio), `sndLose` (saw thud), `sndDraw` (square blip), `sndMatchWin` (chord), `sndMatchLose` (descending saw).

### Persistence
`localStorage` key **`prs_best`** — best endless win streak only. `loadBest()` on init, `saveBest()` when a new best is reached. Best-of match results are session-scoped and not persisted.

### Language support (en / ar)
- `STRINGS.en` / `STRINGS.ar` — all visible text including pause/resume/reset labels. Some entries are functions: `bestOf(n)`, `best(n)`.
- `applyOverlayStrings()` — applies active language to the overlay; toggles `#overlay.rtl`.
- `updateLangButton()` — syncs `.lang-btn` active state, sets `<html lang>`, calls `applyOverlayStrings()` + `syncGameBtns()`.
- When adding a new string: add to **both** `STRINGS.en` and `STRINGS.ar` in parallel, then reference via `STRINGS[state.language].myKey`.

### Visual / CSS
- All colors are CSS custom properties on `:root` (`--accent`, `--win`, `--lose`, `--draw`, `--ink`, `--card`, …) — retheme from one place.
- Responsive: `@media (max-width:520px)` compacts HUD, hides `.gb-label`/`.gb-key` on game buttons, hides `.mv-key` on move buttons; `@media (max-height:560px)` tightens the arena.
- `viewport-fit=cover` + `env(safe-area-inset-*)` padding for notched phones.
- **Do not use `background-attachment: fixed`** — it causes iOS Safari repaint jank and is broadly unsupported on mobile WebKit.
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
| `R` / `P` / `S` (or `1`/`2`/`3`) | Playing | Throw Rock / Paper / Scissors |
| `Esc` | Playing | Pause |
| `Esc` / `Enter` / `Space` | Paused | Resume |
| `Enter` | Start or game-over overlay | Start / Play Again |

---

## Handoff Notes

> Read this section when picking up the project after a gap, or handing it to a new collaborator. It captures the current state, the reasoning behind non-obvious choices, established conventions, and the concrete next steps with enough detail to implement them without re-deriving context.

### Current project state (as of 2026-06-30)

The game is **complete and live**. Every feature described in this file works. The file is ~780 lines of self-contained HTML/CSS/JS. Deployment: GitHub `mo-io/prs-game` (private) → Vercel auto-deploy on push to `master`.

**What is fully working:**
- Classic RPS: vs fair-random CPU, correct win/lose/draw resolution
- Best-of-3/5/7 and Endless match modes
- Pause / Resume / Reset in-game (Pause gated by `state.locked` — no mid-animation interrupts)
- Shake→Reveal→Resolve round animation (~760ms)
- Procedural Web Audio (6 distinct sound events)
- Full English/Arabic bilingual UI with RTL support
- Mobile responsive (≥74px tap targets, safe-area padding, compact HUD at ≤520px)
- `prs_best` localStorage streak persistence
- `#game-btns` (Pause + Reset) visible only during active play
- `#pause-overlay` with Resume + Reset, semi-transparent

### Key decisions and why

**Single self-contained HTML file**
Same pattern as sibling project Word Strike. Zero dependencies, no build tooling, Python HTTP server only. Never split into multiple files — the entire game must remain in `prs-game.html`.

**Win rules as data, not logic**
`MOVES`, `BEATS`, `GLYPH`, `beats()` are the only things that determine outcomes. This was explicitly designed for Lizard-Spock: extending the game later requires only adding array/object entries and HTML buttons — no if-branches, no switch statements. Do not add `if (move === 'lizard')` conditionals.

**"Arcade Duel" theme (not neon)**
The user explicitly chose a distinct visual identity separate from Word Strike's neon cyberpunk. The indigo/violet palette with semantic color vars (`--win`, `--lose`, `--draw`) is intentional. Do not introduce `--neon-*` variables or glow-heavy styling.

**Endless-only streak persistence**
Only Endless mode writes to `prs_best`. Best-of-N results are session-scoped. Reasoning: Best-of has a clear match result already; persisting it would require a separate format. Endless "best streak" is the natural long-term metric. If you add Best-of persistence, use a separate `localStorage` key.

**Move buttons as primary input (no hidden input)**
Word Strike uses a hidden `<input>` to trigger the iOS virtual keyboard. PRS-Game doesn't need text entry — the three buttons are the entire interface. Do not add a hidden input unless a future feature genuinely requires text (e.g., a player-name field).

**Pause gated by `state.locked`**
`pauseGame()` returns early if `state.locked` is true (i.e., during the ~760ms reveal animation). The round is short enough that waiting for it to complete is acceptable UX. This avoids the complexity of cancelling and replaying in-flight `setTimeout` timers.

**No `background-attachment: fixed`**
Removed during initial dev. This CSS property causes iOS Safari repaint jank (poorly supported on mobile WebKit) and also stalls the headless preview screenshot tool. Keep backgrounds as simple gradients without this attachment.

### Established coding conventions

**Adding any new UI string:**
1. Add to `STRINGS.en` and `STRINGS.ar` simultaneously (never one without the other)
2. Reference as `STRINGS[state.language].myKey` inside functions that read strings
3. Call `syncGameBtns()` or `applyOverlayStrings()` (whichever is appropriate) after language changes so the new string renders correctly

**Adding a new lifecycle state transition:**
1. Update `state.running`, `state.paused`, `state.locked` as appropriate
2. Show/hide `#overlay`, `#pause-overlay`, `#game-btns` with `.classList.add/remove('hidden')`
3. Add/remove `disabled` class from `#controls`
4. Call `syncGameBtns()` + `updateHUD()` at the end

**Adding a new sound:**
Use `playTone(freq, type, dur, vol, delay)` — do not create a new oscillator setup from scratch. Chain multiple `playTone()` calls with staggered `delay` values for arpeggios. Wrap in a named `const sndXxx = () => ...` function.

**Adding a new CSS component:**
Define colors exclusively via existing `--var` references from `:root`. Never hardcode hex colors in component rules. Add new semantic vars to `:root` if needed. Follow the mobile-first compact rule: add a `@media (max-width:520px)` override for anything that needs to shrink or hide on mobile.

**DOM shorthand:** `const $ = id => document.getElementById(id)` — use `$('id')` for ID lookups. Use `document.querySelector` / `document.querySelectorAll` for class or attribute selectors.

**`.hidden` pattern:** All overlays and conditional panels use `.hidden { display:none }`. Toggle with `.classList.add/remove('hidden')`. Do not toggle `style.display` directly in JS for these elements.

### Files that need attention

| File | Why |
|------|-----|
| `prs-game.html` | The only code file — all features go here |
| `MEMORY.md` | Update "Current State" and "Build History" after each feature addition |
| `CLAUDE.md` | Update "Keyboard shortcuts", "State machine", and "Lifecycle functions" tables when they change |

`MEMORY.md` and `CLAUDE.md` are not auto-updated. After shipping a feature, update both manually before committing.

### Specific next steps (implementation-ready)

#### 1. Lizard-Spock mode (highest value / lowest effort)

The data model already supports it. This is purely additive.

**Step 1 — extend the data:**
```js
const MOVES = ['rock','paper','scissors','lizard','spock'];
const BEATS = {
  rock:    ['scissors','lizard'],
  paper:   ['rock','spock'],
  scissors:['paper','lizard'],
  lizard:  ['spock','paper'],
  spock:   ['scissors','rock'],
};
const GLYPH = { rock:'✊', paper:'✋', scissors:'✌️', lizard:'🦎', spock:'🖖' };
```

**Step 2 — add two move buttons** in `#controls`, matching the existing `.move-btn` structure exactly:
```html
<button class="move-btn" data-move="lizard"><span class="mv-glyph">🦎</span><span class="mv-name">Lizard</span><span class="mv-key">L</span></button>
<button class="move-btn" data-move="spock"><span class="mv-glyph">🖖</span><span class="mv-name">Spock</span><span class="mv-key">K</span></button>
```

**Step 3 — add strings** to both `STRINGS.en` and `STRINGS.ar`:
```js
// en: lizard:'Lizard', spock:'Spock'
// ar: lizard:'السحلية', spock:'سبوك'
```

**Step 4 — add keyboard mappings** in `KEYMAP`: `l:'lizard', k:'spock'`

**Step 5 — mode toggle on start overlay.** Add a segmented control `#variant-seg` alongside `#mode-seg` with two options: "Classic" (3 moves) and "Lizard-Spock" (5 moves). Toggling it shows/hides the lizard/spock buttons via a `.ls-only` CSS class. Store selection in `state.variant = 'classic' | 'ls'`.

**CSS note:** With 5 buttons, `#controls` will need `flex-wrap:wrap` or smaller `max-width` per button on mobile. Target `.move-btn` at `@media (max-width:520px)` with `min-height:64px; max-width:calc(33% - 8px)` for 3-per-row layout.

---

#### 2. Mute / sound toggle (low effort)

**Step 1** — add `muted: false` to `state`. Load from `localStorage` key `prs_muted` on init.

**Step 2** — gate `playTone()`:
```js
function playTone(freq, type='sine', dur=0.06, vol=0.12, delay=0){
  if (!state.audioCtx || state.muted) return;
  // ... rest unchanged
}
```

**Step 3** — add a mute button to `#game-btns` (or the HUD):
```html
<button class="game-btn" id="btn-mute"><span class="gb-icon">🔊</span></button>
```
Toggle: `state.muted = !state.muted; localStorage.setItem('prs_muted', state.muted); syncGameBtns();`

**Step 4** — update `syncGameBtns()` to set the icon to `🔇` when muted.

**Step 5** — add `STRINGS.en/ar` entries: `mute:'Mute'`, `unmute:'Unmute'`.

---

#### 3. Streak milestone celebrations (low effort)

In `resolveRound()`, after updating `state.streak`, add:
```js
if (state.mode === 'endless' && state.streak > 0 && [5,10,20,50].includes(state.streak)){
  showMilestone(state.streak);
}
```

`showMilestone(n)` — create/reuse a `#milestone-banner` div (similar to `#banner`), briefly animate it in/out with a CSS class, then remove. Example content: `🔥 ${n} streak!` / Arabic equivalent.

---

#### 4. Match-point alert (low effort)

In `resolveRound()`, after checking match end, add:
```js
const target = Math.ceil(state.bestOf / 2);
if (state.mode === 'bestof'){
  if (state.youScore === target - 1) youCard.classList.add('match-point');
  if (state.cpuScore === target - 1) cpuCard.classList.add('match-point');
}
```

Add CSS:
```css
.glyph-card.match-point {
  border-color: var(--draw);
  animation: pulse-border 0.8s ease-in-out infinite;
}
@keyframes pulse-border {
  0%,100%{ box-shadow: 0 0 0 2px rgba(255,193,69,.3), var(--shadow); }
  50%    { box-shadow: 0 0 0 6px rgba(255,193,69,.6), var(--shadow); }
}
```

Clear `.match-point` in `resetCards()`.

---

#### 5. Predictive AI (moderate effort)

**Step 1** — add `aiMode:'random'` and `moveHistory:[]` to `state`. Reset `moveHistory` in `startGame()`.

**Step 2** — smart pick function:
```js
function cpuPickSmart(){
  if (state.moveHistory.length < 5) return cpuPick(); // not enough data
  const counts = {};
  MOVES.forEach(m => counts[m] = 0);
  state.moveHistory.slice(-10).forEach(m => counts[m]++);
  const likelyPlayer = Object.keys(counts).reduce((a,b) => counts[a]>counts[b]?a:b);
  // find what beats the player's most frequent move
  const counter = MOVES.find(m => BEATS[m].includes(likelyPlayer));
  return counter || cpuPick();
}
```

**Step 3** — in `playRound()`, push player's move to history: `state.moveHistory.push(move);`

**Step 4** — in `playRound()`, pick based on `state.aiMode`:
```js
const cpu = state.aiMode === 'smart' ? cpuPickSmart() : cpuPick();
```

**Step 5** — add a difficulty segmented control on the start overlay (Easy / Hard), storing `state.aiMode`. Add to `STRINGS.en/ar`: `difficulty:'DIFFICULTY'`, `easy:'Easy'`, `hard:'Hard'`.
