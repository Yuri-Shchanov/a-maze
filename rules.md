# A Maze (Go) — Game Rules (v0)

**Status:** agreed baseline for engine + protocol.  
**Last updated:** 2026-01-28

This document fixes gameplay rules to avoid engine-breaking ambiguities.

---

## 0. Overview

A Maze is a turn-based exploration game played on a hidden opponent maze.

- **Modes:** PvP (2 players) and PvE (player vs bot).
- **Board size:** 10×10 cells.
- **Visibility:** no sensors. Players only learn the result of their own actions and update their own knowledge map.
- **Server authority:** the server stores the full truth (mazes, item placement, treasure truth, effects, positions) and validates actions.  
  The client never receives the full opponent maze.
- **Event log:** the server records an append-only event log sufficient to replay the match.

**Maze creation:** Each player builds their own 10×10 maze. The maze may be created manually, generated procedurally, or produced by any combination of both, as long as it satisfies maze validity rules (defined elsewhere).

---

## 1. Grid & Walls Model (Important)

### 1.1 Cells vs Walls
The maze consists of a **10×10 grid of cells**.

**Walls are located *between* cells**, not inside cells.  
Each cell has four sides (N/E/S/W). Each side has a state:

- `Open` — movement to the adjacent cell is allowed
- `InnerWall` — blocked by an internal wall between two cells
- `OuterWall` — the outer boundary of the maze (blocked)
- `Exit` — an opening on the outer boundary that allows leaving the maze

### 1.2 Invariants
To keep the maze consistent:

1. **Symmetry between neighbors**  
   If cell A’s side toward cell B is `Open`/`InnerWall`, then cell B’s opposite side is the same (`Open`/`InnerWall`).
2. **Outer boundary**  
   Perimeter sides are `OuterWall` by default, except where explicitly marked as `Exit`.
3. **Exit placement**  
   `Exit` is only valid on a perimeter side and always points outward (there is no neighboring cell “outside”).
4. **Cell contents**  
   Items/treasures/pits exist **in cells**, not on edges.

---

## 2. Terminology

- **Action** — an atomic player command. In v0: `Move(dir)` where `dir ∈ {N,E,S,W}`.
- **Turn** — one executed Action (one `Move` attempt).
- **TurnSlot** — one time the turn order grants a player the right to act.  
  A TurnSlot can contain 0, 1, or 2 Turns depending on effects (Trap may force 0).
- **Effect** — a temporary state that changes turn economy (Crutch, Crossbow, Trap).
- **PlayerKnowledge** — the player’s partial map of the opponent maze.
- **KnowledgeIsland** — a connected fragment of knowledge (a local coordinate system). Multiple islands may exist due to teleports.
- **Active Island** — the island used to interpret current exploration after a teleport (or at start).

---

## 3. Turn Order & Turn Economy

### 3.1 Base order
TurnSlots strictly alternate:

`A, B, A, B, ...`

### 3.2 Turns per TurnSlot
Default: each TurnSlot contains **1 Turn** (one `Move`).

Effects may change this:

- **Crutch (Speed-Up):** the owner performs **2 Turns** in each of their TurnSlots.
- **Crossbow (Slow-Down):** the owner is “crippled”; the **opponent** performs **2 Turns** in each of the opponent’s TurnSlots.  
  (The Crossbow owner still performs 1 Turn in their TurnSlots.)

**Crutch and Crossbow cancel each other:**
- If a player currently has one of these effects and gains the opposite one, **both are removed**. No stacking.

### 3.3 Trap (Skip TurnSlots)
Trap applies `SkipSlots = 3`.

While `SkipSlots > 0`:
- when the player would receive a TurnSlot, they perform **0 Turns**,
- the TurnSlot is consumed,
- `SkipSlots -= 1`.

Trap has priority over speed effects:
- if a TurnSlot is skipped, no Turns occur inside it even if Crutch/Crossbow would otherwise grant extra Turns.

---

## 4. Movement (Action Resolution)

### 4.1 Action
`Move(dir)` attempts to move from the current cell through the chosen side.

Resolution depends on the edge state:

- `Open` → **Success:** player moves into the adjacent cell.
- `InnerWall` → **Wall (Inner):** player stays; they learn an inner wall exists in that direction.
- `OuterWall` → **Wall (Outer):** player stays; they learn it is the maze boundary in that direction.
- `Exit` → **Exit:** player leaves the maze (see section 5).

Movement is cardinal only (N/E/S/W). No diagonals.

### 4.2 Cell contents on entry
If movement succeeds into a cell, its contents resolve immediately (treasure/item/trap/pit), in the same Turn.

---

## 5. Exits: Discovery, Leaving, Returning

### 5.1 What an exit is
An **exit is not a special cell**. An exit is an **edge state `Exit` on a perimeter side**.

A player can only leave the maze by:
1) standing on a perimeter cell, and  
2) performing `Move(dir)` outward, where that side is `Exit`.

If the side is `OuterWall`, the player does not leave.

### 5.2 Exits count (v0)
There are **4 exits total**: one on each outer side (North, South, West, East).  
(Implementation should allow changing the number/rules later.)

### 5.3 Start cell and exit interaction
A player **cannot “start on an exit”** because exits are edges, not cells.

To learn an exit location, the player must discover it by attempting to move outward from a perimeter cell.

### 5.4 Return after leaving
If a player leaves the maze and the match does not end:
- the player returns to the **same perimeter cell** they exited from (the cell adjacent to the exit edge).
- return happens automatically before the player’s next participation in turn order and does not consume a Turn.

### 5.5 Constraint near exits
To avoid semantic conflicts:
- A cell that has an `Exit` edge must not contain `PitIn` (teleport entry).

---

## 6. Treasures & Win Conditions

### 6.1 Treasure set
There are **4 treasures** in each maze:
- 1 **True** treasure
- 3 **Fake** treasures

### 6.2 Finding a treasure
When stepping onto a treasure cell, the player only learns: **“treasure found”**.  
True/Fake is **not** revealed at that time.

### 6.3 Truth check
Treasure truth is checked **only when the player leaves the maze** via an exit.

- If the player leaves with the **True** treasure found → this satisfies the treasure requirement.
- If the player leaves with a **Fake** treasure → no win; the player must return and continue searching.

### 6.4 Win condition
A player wins if:
1) they have found the **True** treasure, and  
2) they subsequently **leave the maze** via an exit.

### 6.5 Tie
If both players satisfy the win condition in the same match resolution window, the result is a **draw** (no extra turns).

---

## 7. Items & Effects

All items are hidden until stepped on (unless already known via the player’s own knowledge).

### 7.1 Crutch
On entering a Crutch cell:
- if the player has Crossbow → remove Crossbow and do **not** keep Crutch (both cleared).
- else → apply **Crutch**.

### 7.2 Crossbow
On entering a Crossbow cell:
- if the player has Crutch → remove Crutch and do **not** keep Crossbow (both cleared).
- else → apply **Crossbow**.

### 7.3 Trap
On entering a Trap cell:
- set `SkipSlots = 3` (see §3.3).  
(If Trap is triggered again while already active, it does not stack; v0)

---

## 8. Teleports (PitIn/PitOut) & Knowledge Islands

### 8.1 Pits
There are 4 pairs of pits:
- `PitIn #k` (entry) and `PitOut #k` (exit), for `k ∈ {1..4}`.

### 8.2 Triggering a teleport
Entering `PitIn #k` causes an immediate teleport to `PitOut #k` in the same Turn.

### 8.3 What the player knows after teleport
After teleport, the player **does not know** their global position relative to previously learned areas.

Important clarification:
- The player’s local map is never “wrong” — they are still in the same maze.
- Teleporting only breaks the ability to **align** newly learned information with previously learned areas.

### 8.4 Knowledge islands
After teleport:
- create a new **KnowledgeIsland** and set it as **Active Island**.
- previously known areas remain as other islands (including the “global/main” island).

---

## 9. Merging Knowledge via Position Hypothesis (Anti-Bruteforce)

### 9.1 When merge is available
Merge is available only when the player has:
- a main/global island, and
- an additional Active Island (created by teleport).

If there is no additional island, merge is not offered (nothing to merge).

### 9.2 When merge can be attempted
Before executing an Action inside a TurnSlot, the player may attempt **at most one** merge attempt.

- Merge attempt does **not** consume a Turn.
- Regardless of success/failure, the attempt is **spent** for that TurnSlot.

### 9.3 What a merge attempt is
The player makes a hypothesis: **“My current position corresponds to cell X in the main/global map.”**

Server validates compatibility (overlap constraints, known walls, known contents where applicable).

### 9.4 Outcomes
- **Success:** islands are merged; the merged area becomes the Active Island; the previous Active Island is removed as a separate island.
- **Failure:** no knowledge changes. The player continues exploring within the Active Island. No second attempt this TurnSlot.

Failure does not create a “wrong map”; it simply means no alignment was established.

---

## 10. Starting Position

### 10.1 Choosing a start cell
The attacker (exploring player) chooses a start **cell** in the opponent maze.

- The player explicitly knows which cell they selected (this is the origin of their first KnowledgeIsland).
- Since exits are edges, selecting a start cell does not place the player “on an exit”.

### 10.2 Immediate reveal & resolution
The start cell is immediately marked **visited** in PlayerKnowledge.

If the start cell contains any content:
- items/traps resolve immediately.
- (Teleport resolution is treated like stepping onto the cell, but note: cells adjacent to Exit edges must not contain `PitIn`.)

---

## 11. Server Authority & Information Flow (Summary)

Server owns truth:
- maze geometry, exit edges
- item/treasure/pit placement
- treasure truth
- player true positions
- effects and counters
- event log

Client receives:
- action result (success/wall(inner|outer)/object/exit/etc.)
- knowledge updates (visited cells, discovered walls/exits, “treasure found”)
- effect changes that are observable for that player (their own effects; optionally opponent tempo effects only insofar as it changes the number of turns they get)

---

## 12. Notes / TBD (explicitly not fixed yet)

- Exact maze validity constraints for generator/editor (reachability guarantees).
- Whether items are consumed on pickup (v0 assumes yes; define in engine).
