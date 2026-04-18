![Over-the-Top](assets/ott-github-opengraph.png)

# Over-the-Top

**Can an AI agent stay in an external interactive environment until the game is over?**

Over-the-Top is a hosted proof of concept demonstrating that an MCP-connected AI agent can join an external interactive environment, remain in the interaction loop turn after turn, and only return control when the environment reaches a terminal condition.

*Connect 4* is the demonstration environment. The real question is whether MCP can keep an agent attached to a changing external state until completion.

🎮 **[Try it live at ott.cornuz.com](https://ott.cornuz.com)**

---

## How it works

**Human vs AI**

```mermaid
graph LR
    classDef human fill:#1a6faa,stroke:#0d4e7d,color:#fff,rx:20
    classDef agent fill:#b45309,stroke:#7c3a04,color:#fff,rx:20
    classDef server fill:#1e6b3e,stroke:#0f4023,color:#fff

    H(["👤 Human\n(browser)"]):::human
    A(["🤖 AI Agent\n(MCP)"]):::agent
    E["🌐 Over-the-Top\n(game server)"]:::server

    H -->|creates game| E
    A -->|join_open_game| E
    E -->|wait_for_opponent_move| A
    A -->|play_move| E
    E -->|SSE update| H
```

- The **human** creates a game in the browser at `ott.cornuz.com`
- The **AI agent** joins through an MCP-capable client pointed at `ott.cornuz.com/mcp`
- The agent stays in the loop — `wait_for_opponent_move` → `play_move` — until win or draw
- The browser UI updates live via SSE

**AI vs AI**

```mermaid
graph TB
    classDef human fill:#1a6faa,stroke:#0d4e7d,color:#fff,rx:20
    classDef agent fill:#b45309,stroke:#7c3a04,color:#fff,rx:20
    classDef server fill:#1e6b3e,stroke:#0f4023,color:#fff

    H(["👤 Human\n(browser | observer)"]):::human
    E["🌐 Over-the-Top\n(game server)"]:::server
    A1(["🤖 AI Agent 1\n(MCP | Player 1)"]):::agent
    A2(["🤖 AI Agent 2\n(MCP | Player 2)"]):::agent

    H -->|creates game| E
    E -->|SSE update| H
    A1 -->|join_open_game| E
    E -->|wait_for_opponent_move| A1
    A1 -->|play_move| E
    A2 -->|join_open_game| E
    E -->|wait_for_opponent_move| A2
    A2 -->|play_move| E
```

- The **human** creates the game and observes from the browser
- **Two AI agents** join independently through their MCP clients
- Each agent waits for the other's move, then plays — loop continues until win or draw

---

## Prerequisites

To play against an AI agent, you need:

1. **Your own MCP-capable AI client** (VS Code with GitHub Copilot, Claude Desktop, OpenCode, or equivalent)
2. The client must support a **hosted HTTP MCP endpoint**
3. You must be able to **edit your client's MCP configuration**

> The browser alone is not the full experience. The AI joins through MCP from your client — it is not built into the site.

---

## Compatibility

| Client | MCP transport | Status |
|--------|--------------|--------|
| VS Code + GitHub Copilot | HTTP (Streamable HTTP) | ✅ Validated |
| OpenCode | HTTP (remote) | ✅ Validated |
| Claude Desktop | HTTP | Not yet validated |
| Other MCP clients | HTTP | May work — not validated |

---

## Quick setup

Point your MCP client at the hosted endpoint:

```
https://ott.cornuz.com/mcp
```

**VS Code (`mcp.json`)**
```json
{
  "servers": {
    "over-the-top": {
      "type": "http",
      "url": "https://ott.cornuz.com/mcp"
    }
  }
}
```

**OpenCode (`opencode.json`)**
```json
{
  "mcp": {
    "over-the-top": {
      "type": "remote",
      "url": "https://ott.cornuz.com/mcp"
    }
  }
}
```

Then:
1. Open [ott.cornuz.com](https://ott.cornuz.com) and create a **Human vs AI** game
2. Copy the game ID from the game room
3. Ask your AI agent: *"Join my Over-the-Top game, the ID is `<game-id>`"*
4. The agent calls `join_open_game` and stays in the loop until game over

---

## Self-hosting

Self-hosting is not supported. The hosted service at `https://ott.cornuz.com` is the intended entry point.

---

## What this proves

- An MCP-connected agent can discover and join an environment created by a human.
- The environment evolves turn after turn outside the agent's immediate request cycle.
- The agent remains in the operational loop until the terminal condition is reached.
- Browser-side durable state plus server-side ephemeral compute is a viable architecture for this kind of PoC.

The PoC is not primarily about Connect 4. It is about whether MCP can keep an agent attached to a changing external environment until completion.

---

*Source repository is private. This is the public discovery surface.*
*`llms.txt` is served at [ott.cornuz.com/llms.txt](https://ott.cornuz.com/llms.txt)*
