# PRS-Game — Development Memory

Running log of what has been built, decisions made, and what comes next. Read this alongside `CLAUDE.md` (which covers architecture and how to work with the code).

---

## Current State (2026-06-30)

The game is fully playable on **desktop and mobile** in both **English and Arabic**. It is live on GitHub (`mo-io/prs-game`, private) and auto-deploys to Vercel from `master`.

### What works
- Classic RPS and **Lizard-Spock** (5-move) variant — switchable via VARIANT control on start overlay
- **Best-of-N** match modes (3 / 5 / 7) — first to `ceil(N/2)` round wins ends the match
- **Endless** mode — never auto-ends; tracks current win streak and persists best streak to `localStorage` (`prs_best`)
- "Shake → Reveal → Resolve" round animation (CSS keyframes, no game loop)
- Result banner (You win! / CPU wins! / Draw) with win/lose/draw color coding
- Full procedural Web Audio: tick (shake), ascending arpeggio (round win), sawtooth (round lose), blip (draw), chord (match win), descending saw (match lose)
- HUD: YOU score · mode badge (Best-of target or live streak) · best-streak chip · CPU score
- Keyboard shortcuts: R/P/S (or 1/2/3) to throw, L/K for Lizard/Spock (LS mode only), Enter to start/play again, Esc to pause
- **Mobile**: ≥64px tap targets, 3+2 grid in LS mode, responsive compact HUD/cards at ≤520px, `safe-area-inset` padding for notched phones
- **Arabic language**: full bilingual UI including all LS move names and rule descriptions, RTL text direction
- **Solar Flare** color theme (amber/burnt-orange) — active palette; 14 CSS vars in `:root`

---

## Build History

### Lizard-Spock + Solar Flare palette (2026-06-30)
- Added Lizard-Spock variant: `MOVES_CLASSIC`/`MOVES_LS` split, `BEATS` extended to all 10 relationships, `GLYPH` extended, `state.variant`, `updateVariant()`, `#variant-seg` on start overlay, `.ls-only` CSS class, keyboard keys L/K, full Arabic translations including `instructionsLS`
- Mobile LS layout: `#controls.ls-active` wraps to 3+2 grid, `.mv-name` hidden in LS+mobile
- Keyboard handler gated by `MOVES.includes()` — LS keys silently ignored in Classic mode
- Applied **Solar Flare** palette (amber/burnt-orange) replacing Arcade Duel (indigo/violet); theme is 14 CSS vars in `:root`

### Initial release (2026-06-30)
- Single-file architecture (`prs-game.html`) — no build step, no dependencies, Python `http.server 3939` only
- "Arcade Duel" theme: indigo/violet gradient, bold rounded cards, large emoji glyphs (✊✋✌️), semantic `--win`/`--lose`/`--draw` CSS vars — deliberately **not** Word Strike's neon cyberpunk look
- Win logic stored as pure data (`MOVES`, `BEATS`, `GLYPH`, `beats()`) for Lizard-Spock extensibility
- `cpuPick()` — uniform random, no streak manipulation
- Both Best-of-N and Endless modes on the start overlay (segmented control)
- Only Endless mode contributes to the persisted `prs_best` streak; Best-of results are session-scoped
- `state.locked` blocks input during the shake→reveal animation
- Procedural audio: `playTone(freq, type, dur, vol, delay)` helper + lazy `initAudio()` on first user gesture
- Full `STRINGS.en` / `STRINGS.ar` i18n, `applyOverlayStrings()`, `#overlay.rtl` toggle — same pattern as Word Strike
- Move buttons are the primary input (no hidden input hack needed — simpler than Word Strike)
- GitHub repo: `github.com/mo-io/prs-game` (private, SSH auth, account `mo-io`)
- Vercel deploy: connected via dashboard, auto-deploys from `master`
- **`background-attachment: fixed` removed** — causes iOS Safari repaint jank and unsupported on many mobile browsers

---

## Known Gaps / Future Ideas

### Gameplay
- **Predictive / difficulty-selectable AI** — currently always random; a pattern-tracking AI (e.g. frequency analysis of last N moves) was explicitly deferred
- **Local 2-player** — two humans on one device; deferred from initial scope
- No animation for a "match point" alert (one win away from clinching)
- No shake/vibration feedback on mobile for losing a round

### UX / Polish
- No settings screen (sound on/off)
- No share / social feature after a match
- No streak milestone celebrations (e.g., at 5-streak)
- Game-over overlay for Best-of shows raw score; a visual "YOU WIN / CPU WINS" graphic would be more impactful
- No keyboard shortcut for restarting from the game-over overlay (Enter works for Play Again but a dedicated key would be clearer)

### Language / Internationalization
- Only English and Arabic; `STRINGS` pattern is in place to add more languages
- Best streak is shared across both languages (single `prs_best` key)

### Technical
- No unit tests; all testing is manual in-browser
- No CI pipeline (Vercel deploys on push but no automated test gate)
