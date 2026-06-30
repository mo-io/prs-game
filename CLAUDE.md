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

`vercel.json` rewrites `/` → `/prs-game.html` so the root URL serves the game. No GitHub remote / Vercel project is wired up yet — that's a follow-up (mirror Word Strike's setup when wanted).

## Game overview

Two fighter cards face off ("YOU" vs "CPU"). The player throws **Rock ✊ / Paper ✋ / Scissors ✌️** via the three move buttons, the `R`/`P`/`S` keys, or `1`/`2`/`3`. Both cards play a "Rock, Paper, Scissors, Shoot!" shake animation, then reveal the chosen glyphs and resolve the round. A neon-free result banner shows **You win! / CPU wins! / Draw**.

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
To add Lizard-Spock: extend `MOVES`, `BEATS`, `GLYPH`, add two move buttons + their `STRINGS` labels. Nothing else changes.

`cpuPick()` returns a uniform-random move — fair, no streak manipulation.

### State machine
`state = { language, running, locked, mode, bestOf, youScore, cpuScore, round, streak, bestStreak, audioCtx }`.
- `running` — true while a match is in progress (overlay hidden).
- `locked` — blocks input during the reveal animation so a round can't double-fire.
- `mode` — `'bestof'` | `'endless'`; `bestOf` — 3/5/7.
- `overlayMode` (module-level) — `'start'` | `'gameover'`; tells `applyOverlayStrings()` which text set to render. `lastResult` (`'win'`/`'lose'`) colors the game-over title.

### Lifecycle functions
- `startGame()` — inits audio, resets scores/round/streak, hides overlay, enables controls.
- `playRound(move)` — locks input, picks CPU move, runs the shake→reveal animation, then calls `resolveRound()`.
- `resolveRound(result)` — updates scores/streak, banner, sound; checks best-of match end; otherwise unlocks for the next round after a short delay.
- `endMatch(result)` — stops the match, shows the game-over overlay.
- `resetGame()` — back to the start overlay (bound to `Esc`).
- `updateHUD()` — refreshes scores, mode badge, best-streak chip. Call after every score/mode change.

### Round flow / animation
No persistent game loop. A round is driven by `setTimeout`/`setInterval`: both cards show ✊ with the `shake` CSS class while `sndTick()` fires 3 times, then after ~760ms the real glyphs are set with a `pop` animation and the outcome resolves. CPU card is mirrored (`scaleX(-1)`) so the fists face each other — its shake/pop keyframes have `-cpu` variants to preserve the mirror.

### Audio
Procedural via Web Audio (`state.audioCtx`), lazily created in `initAudio()` on first interaction (autoplay policy). All sounds are synthesized by `playTone(freq, type, dur, vol, delay)` — same helper as Word Strike. Events: `sndTick` (shake), `sndWin` (ascending arpeggio), `sndLose` (saw thud), `sndDraw` (square blip), `sndMatchWin` (chord), `sndMatchLose` (descending saw).

### Persistence
`localStorage` key **`prs_best`** — best endless win streak. `loadBest()` on init, `saveBest()` whenever a new best is set. Only Endless contributes to the persisted record; Best-of matches are session-scoped.

### Language support (en / ar)
- `STRINGS.en` / `STRINGS.ar` — all visible text (title, sub, instructions HTML, mode labels, banners, move names, stat labels, best-streak line). Some entries are functions, e.g. `bestOf(n)`, `best(n)`.
- `applyOverlayStrings()` — applies the active language to the overlay (start vs game-over via `overlayMode`), HUD labels, duel names, and move-button names. Toggles `#overlay.rtl` for RTL.
- `updateLangButton()` — syncs the active `.lang-btn`, sets `<html lang>`, then calls `applyOverlayStrings()`.

### Visual / CSS
- All colors are CSS custom properties on `:root` (`--accent`, `--win`, `--lose`, `--draw`, `--ink`, `--card`, …) — retheme from one place.
- Responsive: `@media (max-width:520px)` compacts the HUD and hides move-button key hints; `@media (max-height:560px)` tightens the arena.
- `viewport-fit=cover` + `env(safe-area-inset-*)` padding for notched phones. Move buttons are the primary input (≥74px tap targets), so no hidden-input keyboard hack is needed — simpler than Word Strike.

## Keyboard shortcuts
| Key | Action |
|-----|--------|
| `R` / `P` / `S` (or `1`/`2`/`3`) | Throw Rock / Paper / Scissors |
| `Enter` | Start / Play Again (from start or game-over overlay) |
| `Esc` | Reset to start screen (during a match) |

## Out of scope (possible follow-ups)
- Lizard-Spock 5-move mode (structure already supports it; UI not built).
- Local 2-player, or predictive / selectable AI difficulty.
- GitHub remote + Vercel deployment wiring.
