# PRS-Game

Rock · Paper · Scissors — with an optional Lizard · Spock variant. A single self-contained HTML file, no build step, no dependencies.

Play it live: deployed via Vercel from this repo's `master` branch.

## Features

- **Classic** (Rock/Paper/Scissors) and **Lizard-Spock** (5-move) variants, switchable at the start screen
- **Best-of-3/5/7** match mode, or **Endless** mode with win-streak tracking
- Persisted best streak (`localStorage`), Endless mode only
- Keyboard, mouse, and touch controls
- Procedural sound effects (Web Audio, no audio files)
- Full **English / Arabic** bilingual UI with RTL support
- Mobile-responsive layout with safe-area padding for notched phones
- "Solar Flare" amber/burnt-orange visual theme

## Running locally

No Node/npm required — just Python's built-in HTTP server:

```
python -m http.server 3939
```

Then open `http://localhost:3939/prs-game.html`.

## How to play

Two fighters face off — **YOU** vs **CPU**. Throw a move using the on-screen buttons, or the keyboard:

| Key | Action |
|-----|--------|
| `R` / `P` / `S` (or `1`/`2`/`3`) | Rock / Paper / Scissors |
| `L` / `K` | Lizard / Spock (Lizard-Spock variant only) |
| `Esc` | Pause |
| `Enter` | Start / Play Again |

Both cards play a "Rock, Paper, Scissors, Shoot!" shake animation before revealing moves and resolving the round. In Best-of-N, the first player to reach the majority of round wins takes the match. In Endless mode, keep playing to build your streak — your best streak is saved locally.

## Project structure

Everything lives in one file:

```
prs-game.html   — HTML, CSS, and JavaScript for the entire game
vercel.json     — rewrites "/" to "/prs-game.html" for deployment
LICENSE         — MIT license
```

## Architecture notes

The win/lose rules are defined as data (`BEATS`, `GLYPH`), not branching logic, which is what makes the Lizard-Spock variant a data extension rather than a rewrite. See `CLAUDE.md` for full architectural documentation, state machine details, and contributor notes.

## Deployment

Connected to Vercel via the dashboard — every push to `master` auto-deploys. No Vercel CLI needed.

## License

[MIT](LICENSE)
