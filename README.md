# Over-the-Top

[![Build Status](https://github.com/cornuz/over-the-top/workflows/Deploy/badge.svg)](https://github.com/cornuz/over-the-top/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Live Demo](https://img.shields.io/badge/Live%20Demo-ott.cornuz.com-blue)](https://ott.cornuz.com)

**Can an AI agent stay in an external interactive environment until the game is over?**

Over-the-Top is a proof of concept demonstrating that an MCP-connected AI agent can join an external interactive environment (Connect 4), remain in the interaction loop turn after turn, and only return control when the game reaches a terminal condition.

🎮 **[Try it live at ott.cornuz.com](https://ott.cornuz.com)**

## Start Here

**👉 [Open the live demo](https://ott.cornuz.com)** to see the project in action.

To play against an AI agent, you'll need an MCP-compatible client (VS Code with GitHub Copilot, Claude Desktop, or OpenCode). The live site includes setup instructions for connecting your AI client to the hosted MCP endpoint at `https://ott.cornuz.com/mcp`.

![Screenshot: Game Lobby](docs/screenshots/lobby.png)
![Screenshot: Connect 4 Game](docs/screenshots/game.png)

### How It Works

- Humans play through the browser UI
- AI agents connect via MCP tools to join and play games
- Connect 4 serves as the demonstration environment
- The agent stays engaged turn-by-turn until the game ends

For deeper project guidance, see `AGENTS.md`.
For the next strategic phases, see `ROADMAP.md`.
For baseline manual validation, see `REGRESSION.md`.

## What This PoC Proves

- An MCP-connected agent can discover and join an external environment created by a human.
- The environment can evolve turn after turn outside the agent's immediate request cycle.
- The agent can remain in the operational loop until game over.
- Browser-side durable state plus server-side ephemeral compute is a viable architecture for this kind of PoC.
- State can be restored after server restart through `sync_request` and `/api/games/restore`.

## Architecture

- `src/index.ts`: starts the app in stdio mode for MCP clients or in HTTP mode with `--http`.
- `src/server.ts`: creates the MCP server and the Express app; mounts auth, CORS, security, and rate-limiting middleware.
- `src/mcp/tools.ts`: MCP tools used by agents.
- `src/api/routes.ts`: REST routes, long-polling wait endpoint, SSE event stream, and auth-aware lobby endpoint.
- `src/auth/github.ts`: GitHub OAuth, session helpers, `isAdmin()`, `isRealAdmin()`, admin-mask toggle, and `/auth/me`.
- `src/state/game-store.ts`: in-memory game store and event emitter; write-through to SQLite on every mutation.
- `src/state/sqlite-store.ts`: prepared-statement helpers for the SQLite database (games, users tables).
- `src/state/completed-game-archive.ts`: writes completed games to SQLite; migrates JSON archive on startup.
- `src/web/app.js`: browser client, localStorage persistence, SSE reconnect and restore logic, auth status, and lobby visibility.

Key architectural choices:

- Server RAM is ephemeral; browser localStorage is the durable source of in-progress state for the PoC.
- SQLite (`better-sqlite3`) is the persistence layer for game state, user records, and sessions. `OTT_DATA_DIR` sets the database file location (default `./data`).
- Seat types are explicit at game creation time, not inferred later from empty seats. The two seat types are `human/browser` and `ai/mcp`.
- Browser `Human vs AI` creates seat 1 as `human/browser` and seat 2 as `ai/mcp`, with the browser creator immediately occupying the human seat.
- Browser `AI vs AI` creates both seats as `ai/mcp`; the browser stays an observer until MCP/API agents fill those seats.
- SSE is used both for live updates and for restore coordination after server restart.
- Refreshing or reopening the browser lobby can also republish saved `waiting` and `playing` games from localStorage back into server RAM, so MCP clients can discover them again after a restart.
- `join_open_game` is the intended MCP entry point for agent gameplay. For a new waiting game, the human should paste the full game ID from the game room rather than relying on global waiting-game discovery.
- Waiting games expire after 1 hour by default (`OTT_WAITING_GAME_TTL_HOURS`).
- `GET /api/games` is for MCP agents and API tooling. Unauthenticated callers do not receive waiting games globally anymore; logged-in non-admin users only see their own active games; admins still see all games. `GET /api/lobby/games` remains the browser lobby endpoint (auth-aware): anonymous visitors see only completed games; logged-in users see their own active games plus completed; admins see all.
- Out-of-band trace artifacts can be written to `artifacts/regression/` for later offline analysis.
- Optional model elicitation can be enabled for MCP sessions, and its lifecycle is traced so it can be compared against the baseline loop behavior.

## Validated Behavior

- The web UI can create a new Connect 4 game.
- An MCP agent can join the human game through `join_open_game`.
- Explicit seat typing is preserved end to end: `Human vs AI` stays `human/browser` + `ai/mcp`, and `AI vs AI` stays `ai/mcp` + `ai/mcp`.
- `wait-for-move` resolves correctly when a game starts because the missing player joins before the first move.
- Haiku completed a full game through the MCP loop without interruption.
- The browser persists full game state in localStorage.
- The server can recover game state after restart through `sync_request` and `/api/games/restore`.
- SSE keeps the browser UI in sync with moves.
- AI move reasoning can be displayed in the browser.

- GitHub OAuth login and logout flow tested and valid end to end.
- Lobby visibility rules enforced: anonymous visitors see only completed games; logged-in users see their own active games plus completed; admins see all.
- Game delete: logged-in users can delete their own active games; completed games and games owned by others are protected.
- Admin mask: header toggle switches real server-side admin behavior on and off; restore button stays accessible at all times.

Current validated scope:

- topology: 1 human <-> 1 agent
- MCP client runtime: VS Code Copilot

## MCP Setup

To connect an AI agent to the hosted service at `https://ott.cornuz.com/mcp`:

### VS Code with GitHub Copilot

Add this to your `.vscode/mcp.json` (see [`.vscode/mcp.json`](.vscode/mcp.json) in this repo for the full example):

```json
{
  "servers": {
    "over-the-top-prod": {
      "type": "http",
      "url": "https://ott.cornuz.com/mcp"
    }
  }
}
```

For local development, you can also use the stdio configuration shown in the repo.

### OpenCode

Add to your `opencode.json` (see [`opencode.json`](opencode.json) in this repo for the full example):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "over-the-top": {
      "type": "remote",
      "enabled": true,
      "url": "https://ott.cornuz.com/mcp"
    }
  }
}
```

### Claude Desktop

Claude Desktop MCP configuration varies by platform. Check the official MCP documentation for HTTP server setup patterns.

Once connected, ask your AI agent to use the `join_open_game` tool to join a waiting game you've created in the browser.

## Developer Setup

### Local Development

Install dependencies:

```bash
npm install
```

Sync the local Material Symbols Outlined SVG sprite after changing the icon manifest:

```bash
npm run icons:sync
```

Sync the local favicon bundle after updating the RealFaviconGenerator source files:

```bash
npm run favicons:sync
```

`npm run dev` and `npm run build` already run this sync automatically. Keep the generated favicon files under `src/web/` checked into the repository, alongside the SVG logos, because `src/web/` is the browser-served asset root.

Start the HTTP server used by the browser and by MCP tool fetches:

```bash
npm run dev
```

The VS Code MCP client uses `.vscode/mcp.json` and starts the MCP server in stdio mode with:

```bash
npx tsx src/index.ts
```

Optional model elicitation:

- `.vscode/mcp.json` already enables the model-elicitation experiment for the VS Code stdio path with `OTT_ENABLE_MODEL_ELICITATION=1`.
- In HTTP mode, model elicitation is controlled by the environment of the process that runs `npm run dev`.
- PowerShell example:

```powershell
$env:OTT_ENABLE_MODEL_ELICITATION = "1"
npm run dev
```

- Bash or WSL example:

```bash
OTT_ENABLE_MODEL_ELICITATION=1 npm run dev
```

- `opencode.json` only tells OpenCode where the remote MCP endpoint lives. It does not start the HTTP server and it does not inject `OTT_ENABLE_MODEL_ELICITATION` into an already-running Windows server process.
- Even when the server-side feature is enabled, the prompt appears only if the MCP client advertises `elicitation.form` support. If the client does not support it, gameplay continues and the reason should appear in the regression traces as `disabled` or `unsupported`.

Typical local verification flow:

1. Start `npm run dev`.
2. Restart the MCP server in VS Code so it reloads `.vscode/mcp.json` and tool metadata.
3. Open the browser UI at `http://127.0.0.1:3000`.
4. Create a `Human vs AI` or `AI vs AI` game.
5. Ask the agent to play.
6. Confirm the agent enters through `join_open_game` and stays in the loop until game over.

Local origin note:

- Use one canonical browser origin during development: `http://127.0.0.1:3000`.
- Browsers treat `127.0.0.1:3000` and `localhost:3000` as different origins, so their `localStorage` state is separate.
- Because this PoC uses browser `localStorage` as the durable source of in-progress game state, mixing those origins can make the UI appear to have different game sets.
- The server now redirects browser and API traffic from other local loopback aliases to the canonical browser host by default, while leaving `/mcp` untouched.

WSL reachability note:

- The HTTP server intentionally leaves the listen host unspecified.
- On Windows, Node commonly exposes that as an all-interface listener reported as `::`, which is sufficient for the Windows-side browser on `127.0.0.1` and for WSL access through the Windows host IP.
- In other words, the server does not need separate listeners on `127.0.0.1` and `0.0.0.0` for this setup.
- From Windows, use `http://127.0.0.1:3000/` for the browser UI.
- From WSL, if loopback does not reach the Windows host, use `http://<windows-host-ip>:3000/` and `http://<windows-host-ip>:3000/mcp` instead.

## Authentication

GitHub OAuth is the only login method. To enable it, create a GitHub OAuth App and set these environment variables:

```
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_CALLBACK_URL=http://127.0.0.1:3000/auth/github/callback
SESSION_SECRET=change-me
```

Admin access is controlled separately:

```
OTT_ADMIN_GITHUB_IDS=12345678,87654321
```

Set this to the comma-separated numeric GitHub user IDs that should have admin privileges. Admins see all games in the lobby and can delete any game.

Without a GitHub OAuth App configured, the login button is hidden and all users are treated as anonymous visitors.

### Admin mask

Admins can test the non-admin user experience from the page header without a second account. Click **🧪 Test as user** to suppress admin privileges server-side. Click **↩ Restore admin** to undo it. The mask persists across browser close (7-day session cookie) but is always clearable from the header button.

## Environment Variables

All environment variables with their defaults:

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | HTTP listen port |
| `OTT_BIND_HOST` | `0.0.0.0` | Host interface for the published app port in Docker Compose; use `127.0.0.1` behind host-level Caddy in production |
| `SESSION_SECRET` | `dev-secret-...` | Session signing secret — change in production |
| `GITHUB_CLIENT_ID` | — | GitHub OAuth App client ID |
| `GITHUB_CLIENT_SECRET` | — | GitHub OAuth App client secret |
| `GITHUB_CALLBACK_URL` | — | OAuth callback URL (must match App settings) |
| `OTT_DATA_DIR` | `./data` | Directory for the SQLite database file |
| `OTT_ADMIN_GITHUB_IDS` | — | Comma-separated numeric GitHub IDs for admins |
| `OTT_ALLOWED_ORIGINS` | `*` | Comma-separated allowed CORS origins |
| `OTT_CANONICAL_BROWSER_HOST` | — | Canonical host for browser redirect |
| `OTT_WAITING_GAME_TTL_HOURS` | `1` | Hours before an unjoined waiting game expires and is pruned |

Copy `.env.example` to `.env` and fill in the values before running in production.

## Docker

Build and run with Docker Compose:

```bash
docker compose up --build
```

This builds the multi-stage image, mounts a named volume (`ott-data`) at `/data` for the SQLite database, and starts the server on port 3000. Set `OTT_DATA_DIR=/data` in `.env` to point the database at the mounted volume.

Production on `ott.cornuz.com` uses a host-level Caddy reverse proxy for TLS. Keep `OTT_BIND_HOST=127.0.0.1` in production so the Node app is reachable through `localhost:3000` for Caddy but is not exposed directly on the public interface.

The production Caddy config lives in [`deploy/Caddyfile`](deploy/Caddyfile). The VPS-specific Compose file lives in [`deploy/docker-compose.vps.yml`](deploy/docker-compose.vps.yml). The deploy workflow syncs both files to the VPS, backs up the active config, validates Caddy, reloads it, tests HTTP/HTTPS, and restores the backup if the rollout fails.

Useful direct-origin smoke checks from the VPS:

```bash
curl --resolve ott.cornuz.com:80:127.0.0.1 -I http://ott.cornuz.com/health
curl --resolve ott.cornuz.com:443:127.0.0.1 https://ott.cornuz.com/health
```

To build the image alone:

```bash
docker build -t ott .
```



- Web UI icons are standardized on Material Symbols Outlined.
- Icons are consumed as local SVG assets generated from the official Google source, not from remote web fonts.
- The source of truth is [src/web/material-symbols.json](src/web/material-symbols.json).
- Use the official Google icon name in the manifest, then run `npm run icons:sync`.
- Logos and illustrations are separate assets and do not use this icon workflow.

## Favicons

- Favicon source files generated by RealFaviconGenerator live in `sources/favicon/`.
- Sync them into `src/web/` with `npm run favicons:sync`.
- The generated favicon assets in `src/web/` are part of the repository and should stay versioned, like the SVG logos, because `src/web/` is the web asset root served by Express.

## Important Constraints

- Do not add `console.log` or `console.error` inside `src/mcp/tools.ts`.
- In stdio MCP mode, tool-side console output can interfere with protocol behavior or client handling.
- Logging belongs in the HTTP server flow and browser-facing flow instead.
- Regression analysis should prefer out-of-band trace artifacts over MCP chat output alone.
- Smaller models may be less reliable than larger models for long multi-tool loops.

## Future Exploration

- Measure practical limits of long multi-tool loops: action counts, duration, interruption behavior, and model-family reliability.
- Build an MCP action taxonomy and map support across model and client combinations.
- Test MCP behavior in environments beyond VS Code Copilot.
- Explore cloud and web deployment of both the MCP service and the browser-facing server.
- Explore multi-agent and mixed human/agent topologies.

## Contributing

Issues and suggestions are welcome! Please open an issue to discuss any proposed changes before submitting a pull request.

## Support

Enjoying this project? 

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-cornuz-orange?logo=buy-me-a-coffee)](https://buymeacoffee.com/cornuz)

## License

MIT — see [LICENSE](LICENSE)

## Language Policy

All durable project artifacts must be written in English. This includes documentation, code comments, persisted project guidance, and user-facing product copy, unless a task explicitly requires localization.
