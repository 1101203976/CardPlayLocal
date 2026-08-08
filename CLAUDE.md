# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

雀阁 (CardRoomPro) is a real-time online card-room platform: Node.js + Express + Socket.IO backend, Vue 2 (UMD build, no build step) + jQuery frontend. It supports two games — **斗地主 (Doudizhu, 3 players, 1 deck)** and **掼蛋 (Guandan, 4 players, 2 decks)** — with room-based matchmaking, AI bots, spectator mode, letter suggestions (“智囊”), optional MySQL score persistence, and optional Discuz JWT SSO. It runs without a build step; the server serves `static/` directly.

The project is a continuous refactor of an original Doudizhu fork into a multi-game room platform. The two state machines are deliberately written as two separate, parallel modules rather than a shared abstraction.

## Commands

```bash
npm install              # install deps (express, socket.io, socket.io-client, jsonwebtoken, mysql2)
npm start                # = node server.js, listens on PORT (default 8002)
$env:PORT='8012'; node server.js   # Windows: override port
```

No lint or test tooling exists. The only verification is syntax checking and manual rule regression:

```bash
node --check server.js
node --check game.js
node --check guandan-game.js
node --check db.js
node --check static/js/guandan-suggest.js
node --check static/js/effects.js
```

Regression checklist for card-rule changes (no automated tests):
- Guandan sequences `A2345`, `23456`, `10JQKA`; triple-pair (`223344`), double-triple (`333444`), trio-with-pair (`55522`)
- Straight flush beats ≤5-card bomb; 6+ card bomb beats straight flush; rocket (天王炸) beats everything
- Doudizhu: bidding, bottom cards, play, settlement

## Config

Startup config priority: **environment variables > root `config.json` > code defaults**. `server.js` loads `config.json` into `process.env` (without overwriting already-set env vars) so downstream modules like `db.js` read via `process.env.*`. Copy `config.example.json` → `config.json` (gitignored). Key vars: `PORT`, `JWT_SECRET`, `DB_HOST/PORT/USER/PASSWORD/NAME/TABLE_PREFIX`, `DB_DISABLE=1` (memory-only), `SCORE_BASE`.

`JWT_SECRET` resolution, in order: `process.env.JWT_SECRET` → `sso-secret.txt` file → placeholder `change_this_in_production`. The SSO health check at `GET /api/sso/health` returns only a SHA-256 fingerprint, deliberately not the secret.

## Architecture

### Server (`server.js`)

The heart of the app. Sets up Express + Socket.IO, then builds a `GameServer` object whose behavior is attached via `Object.assign(GameServer.prototype, proto)` and `proto` holds the reactive event handlers. Key state owned by each `GameServer`:

- `clients` — connected sockets, each with `{userName, uid, avatarUrl, socket, deskId, posId}`. `posId === 'spec'` marks a spectator.
- `desks` — the room list. Each `desk` has `positions[]` (seat state: `0` empty / `1` seated / `2` ready), `gameType`, `roomCode`, `isPrivate`, and for guandan `guandanLevelRank` / `guandanLevelLabel`.
- `gameDatas[deskId]` — the live `Game` / `GuandanGame` state machine instance for that room.
- `botTimers[deskId]` — pending AI-action `setTimeout`.

Flow: HTTP request → rejected by the `/api` middleware (only GET/HEAD allowed, rate-limited, JSON 404 fallback — the API surface is read-only by design; all score writes happen server-side at game settlement). Socket.IO emits → `proto` handlers mutate desk/position state and broadcast `*_CHANGE` events room-wide or lobby-wide.

On disconnect/lobby-exit, a room with no remaining humans is cleaned up (`cleanupRoomIfEmpty`): AI removed, timers cleared, `game.init()` reset, desk removed.

### Card state machines

- `game.js` — Doudizhu. `Game()` holds `status` (0 not started / 1 bidding / 2 playing / 3 over / 4 re-deal / 5 error), `contextCards`, `contextScore`, `lastCardInfo`, `userScore`, `sumCount` (for 春天/反春 detection). Card values run 3–17 (16=small joker, 17=big joker). Uses `core-validator.js`.
- `guandan-game.js` — Guandan. 108 cards (2 decks), each card carries a `deck` field (0/1) to distinguish duplicates. `SEQUENCE_ORDER = [14,15,3,...,14]` (A, 2 low-wrap). `HEART_TYPE = 2` marks the level-suit “逢人配” wild card; wild logic is core to `validate`. Seats 0/2 are one team, 1/3 the other. Level promotion via `advanceRank` in the server.

Both expose the same interface the server calls: `init()`, `start().getCards()`, `validate(posId, cards)` → `{status, key, type, len, bomb}`, `next(posId, cards)`, `getStatus()`, `getContextPosId()`, `getResult()`, etc.

### AI bots

Bots are server-side timers (`scheduleBotAction`) that act only when the current-turn position is a bot seat. Doudizhu bots use `shouldCallDoudizhuScore` (hand-power heuristic) + `AISuggest.suggest` from `static/js/ai-suggest.js`. Guandan bots use `GuandanSuggest.suggest` from `static/js/guandan-suggest.js` — **the same module the frontend “智囊” button uses**, so server bots and the suggestion feature share one rule engine. Bots auto-reprepare after a round (`rePrepareBots`).

### Database (`db.js`)

Optional MySQL persistence of per-game scores. `db.init()` runs at startup (before `server.js` `init()`), auto-creates tables `<prefix>doudizhu_score` and `<prefix>guandan_score` if missing. Degrades to memory mode when `mysql2` is unavailable, `DB_USER`/`DB_NAME` unset, or `DB_DISABLE=1`. Only JWT-authenticated real players (with `uid`) get scored — guests, bots, spectators don't. `recordResultToDb` computes deltas at settlement (landlord pays 2× on Doudizhu).

### Frontend (`static/`)

Single-page Vue 2 app in `index.html` (all templates inline, no compiled components). Screen state machine via `where` (0 login / 1 hall / 2 room). `parseUrl` reads `?auth=guest` to skip Discuz login. Key JS modules:
- `js/ai-suggest.js` — Doudizhu letter suggestion (also used server-side).
- `js/guandan-suggest.js` — Guandan letter/AI shared rule engine.
- `js/parser.js` — Doudizhu card-type parsing (client-side validation preview).
- `js/effects.js` — sound/effects; mute toggle.
- `css/theme.css` — theme variables (国风 theme); `css/style.css` — layout/components.
- `js/layer/` — vendored popup component; `jquery.min.js`, `vue.min.js` vendored (no npm/ESM).

## Card display conventions

`value` 3–14 = 3…A, 15 = 2, 16 = small joker, 17 = big joker. Face labels via `rankLabel`/`labelValue` (11=J, 12=Q, 13=K, 14=A, 15=2). Guandan dives deeper: `SEQUENCE_ORDER` treats A as both low and high (e.g. `A2345` and `10JQKA` are both valid sequences). When editing card rules, keep client `parser.js`/`ai-suggest.js` and server `game.js`/`guandan-game.js` in agreement — they are hand-synced, not generated.

## Socket protocol

Snapshot of the event names (client→server and server→client) is documented in `README.md` under “常用 Socket 事件”. The server broadcasts `REFRESH_LIST` to the lobby, `*_CHANGE` events to the room, and `GAME_OVER` + `MY_SCORE` at settlement. JWT, if present, is verified in Socket.IO middleware during handshake (`socket.handshake.auth.token`); a failed token sets `socket.tokenError` and triggers an immediate `LOGIN_FAIL`. Same-name duplicate connections are force-disconnected to avoid username deadlock.

## Deployment notes

- Use HTTPS when Discuz SSO is enabled; set a strong random `JWT_SECRET` in production.
- Socket.IO `pingInterval`/`pingTimeout` are tuned for Cloudflare Orange-Cloud proxying (avoid 100s idle cut).
- `.codex-ui-audit/` holds UI audit screenshots/notes — diagnostic output, not part of the runtime.