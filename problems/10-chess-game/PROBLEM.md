# Chess Game

**Difficulty:** Advanced
**Category:** Game Engine + Command Pattern
**Time Box:** 60 min (discussion) / 120 min (machine coding)
**Key Patterns:** Command (move + undo), Strategy (piece move validation), State (game lifecycle)
**Asked at:** Google, Amazon, Microsoft, Adobe, Goldman Sachs

---

## Problem Statement

Design a chess game engine. Two players alternate turns making moves. The system must validate that each move is legal for the piece type, detect check and checkmate, and support undo/redo of moves. The game should track its full move history.

This is the most complex foundation problem — it tests multiple patterns simultaneously and requires careful entity modeling.

---

## Actors

| Actor | Description |
|-------|-------------|
| Player (White / Black) | Takes turns making moves |
| Chess Engine | Validates moves, detects check/checkmate, manages turn flow |
| Board | 8×8 grid of squares; current piece positions |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Board initialized with standard piece positions
- FR2: Players alternate turns (White first)
- FR3: Each move validated for the specific piece type's movement rules
- FR4: Cannot move into check (own king exposed)
- FR5: Detect check — notify player their king is in check
- FR6: Detect checkmate — game over when no valid move escapes check
- FR7: Undo last move (single undo minimum)

**Secondary (implement if time allows):**
- FR8: Redo move after undo
- FR9: Special moves: castling, en passant, pawn promotion
- FR10: Draw conditions: stalemate, 50-move rule, threefold repetition
- FR11: Export game as PGN (chess notation)
- FR12: Timer per player (chess clock)

---

## Non-Functional Requirements

- **Correctness:** No illegal moves accepted; check detection must be accurate
- **Extensibility:** Adding new piece type or rule change should not require rewriting core
- **Performance:** Move validation for all pieces on the board < 1ms

---

## Constraints and Assumptions

- Standard 8×8 board, standard FIDE piece set
- Two human players (no AI opponent)
- No network play — single machine
- Special moves (castling, en passant) are bonus — state so explicitly
- Out of scope: AI engine, network multiplayer, timer enforcement

---

## Good Clarifying Questions to Ask

1. Two-player only, or single-player vs AI?
2. Is undo required? How many levels?
3. Special moves — castling, en passant, pawn promotion?
4. Do we need checkmate detection or just check?
5. Draw conditions?
6. Chess clock / time limits?
7. Should game history be exportable?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **Game** — manages turn flow, game state (ACTIVE/CHECK/CHECKMATE/DRAW), move history
- **Board** — 8×8 grid of Cell objects; knows current piece positions; validates board state
- **Cell** — (row, col); holds a Piece or is empty
- **Piece** — abstract; has color, position; `getPossibleMoves(board): List<Move>`
- **King, Queen, Rook, Bishop, Knight, Pawn** — extend Piece; each implements move generation
- **Move** — from-cell, to-cell, piece, captured piece (null if none)
- **MoveCommand** — wraps Move; `execute(board)` and `undo(board)` — enables Command pattern
- **MoveHistory** — stack of MoveCommand; supports undo/redo
- **MoveValidator** — validates a move won't leave own king in check
- **Player** — name, color (WHITE/BLACK)
- **GameState** — enum: ACTIVE, WHITE_IN_CHECK, BLACK_IN_CHECK, CHECKMATE, STALEMATE, DRAW

</details>
