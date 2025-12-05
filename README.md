# MyDarts

MyDarts is a mobile-friendly darts scorekeeper built with Vue 3 + Vite. It supports count up and zero-one 301 today, with cricket on the roadmap. Use the bundled MVP spec to guide future feature work.

## Features (current implementation)
- Count Up mode: 8-round, 3-throw flow with per-throw scoring
- ZERO ONE 301 mode: 10-round, 3-throw flow with bust handling, single-out finish, and snapshot-based undo that works after game end
- Snapshot undo history to roll back any throw even after finishing a game
- Player setup for 1–4 players with quick naming defaults
- Mobile-first layout with numeric/S/D/T/BULL/MISS inputs

## Roadmap / Spec
- The full MVP requirements (Count Up, Zero-One 301, and Cricket) are documented in [`docs/MVP_SPEC.md`](docs/MVP_SPEC.md). Use this as the source of truth for upcoming work.

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```

### Lint with [ESLint](https://eslint.org/)

```sh
npm run lint
```
