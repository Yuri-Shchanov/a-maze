# A Maze (Go)

A Maze is a turn-based online maze exploration game with strict fog-of-war, server-authoritative logic, and hidden information.

Players blindly explore the opponent’s maze step by step, learning only from the results of their own actions.  
The objective is to escape the maze with the **true treasure**, whose authenticity is revealed only after leaving the maze.

The project focuses on precise game rules, deterministic resolution, and clean separation between game logic and transport.

---

## Key Features

- Turn-based gameplay (PvP and PvE vs bot)
- Online multiplayer with server authority
- Strict fog-of-war (no sensors, no global visibility)
- Hidden treasures (1 true, 3 fake)
- Traps, items, teleports, and turn-economy effects
- Partial knowledge maps with support for multiple “knowledge islands”
- Full event log for replay and dispute resolution

---

## Gameplay Overview (Short)

- Each player creates their own hidden **10×10 maze**
- Players explore the **opponent’s maze** by attempting moves (N/E/S/W)
- Only movement results are revealed (success, wall, exit, object)
- Treasures are indistinguishable until the player exits the maze
- To win, a player must:
  1. Find the **true treasure**
  2. Exit the maze afterwards

If both players satisfy the win condition simultaneously, the result is a **draw**.

👉 **The complete and authoritative rules are defined in [`rules.md`](./rules.md).**

---

## Game Rules

All gameplay mechanics, invariants, and edge cases are formally specified in:

**[`rules.md`](./rules.md)**

This document is the **single source of truth** for:
- movement and wall semantics,
- turn order and turn economy,
- items and effects,
- teleportation and knowledge merging,
- win conditions and ties.

The engine, protocol, and tests **must conform** to `rules.md`.

---

## Architecture Overview

- **Language:** Go (1.22+)
- **Model:** server-authoritative
- **Networking:** HTTP / WebSocket (planned)
- **Clients:** web-based UI
- **Security model:** the client never receives the full opponent maze
- **Reproducibility:** append-only event log allows full match replay

The project is structured to keep:
- core game logic deterministic and testable,
- transport and storage layers independent.

---

## Repository Structure (Planned)

```text
/engine        core game logic (rules-driven)
/protocol      network messages and validation
/server        HTTP / WS server
/client        web UI
/bot           PvE AI logic
/rules.md      authoritative game rules

(Structure may evolve as the project grows.)
Development Status

🚧 Work in progress

Current focus:

    correctness of core mechanics,

    alignment between engine and rules.md,

    extensibility for future rule variations.

This is a gameplay- and engine-first project; visuals and UX come later.

License
MIT