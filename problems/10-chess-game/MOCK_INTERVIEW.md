# Mock Interview Script — Chess Game

**For the interviewer.**

---

## Opening

> "Design a chess game engine. Two players take turns making moves. The system should validate moves, detect check and checkmate, and support undo. Ask any clarifying questions before designing."

| If candidate asks | Answer |
|------------------|--------|
| Two-player or vs AI? | Two human players; no AI |
| Special moves (castling, en passant)? | Secondary — model the core first; call out these as extensions |
| Undo required? | Yes — at least single-level undo |
| Checkmate detection required? | Yes — game must end on checkmate |
| Stalemate/draw? | Secondary |
| Timer? | Out of scope for now |
| Network/multiplayer? | Single machine only |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1:** Piece hierarchy
- Good: `Piece` abstract class with `getPossibleMoves(board): List<Move>` on each subclass
- Bad: One big `validateMove(piece, from, to)` with switch on piece type

**Key signal 2:** Command pattern for undo
- Good: Each move wrapped in `MoveCommand` with `execute()` and `undo()` — stack enables undo
- Bad: "I'd store previous state of board and copy it back" — expensive deep copy approach

**Key signal 3:** Check detection
- Good: "After every move, check if the moving player's king is now in check — if so, reject the move"
- Subtle: "After every move, check if the OPPONENT's king is now in check — that's check detection"

If stuck on piece hierarchy:
> "How would adding a new custom piece type (like a piece that moves like a knight AND a bishop) affect your design?"

If no Command pattern for undo:
> "You have a stack of moves. How would you implement undo?"

If check detection confuses self vs opponent:
> "After White moves, whose king might be in check as a result?"

---

## Extension Probe (~35 min)

> "Add castling — the king and rook can both move in a single turn under specific conditions."

Strong: `CastlingMove extends Move` or a special `MoveCommand`. King and Rook both execute in same command. Undo is also symmetric. Condition validation (king not in check, neither piece has moved) lives in the move validator.

> "Now add move history export in PGN notation."

Strong: Move history is already tracked as MoveCommand stack. Add `toPGN()` method to each MoveCommand. No structural change.

---

## Concurrency Probe (if applicable)

> "Is there a concurrency concern here?"

Expected: No — it's a two-player sequential game, one move at a time. Candidate should recognize this is NOT a concurrency problem (unlike booking or cache). If online multiplayer were added, then network turn synchronization becomes relevant.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Undo, checkmate, turn management, all 6 piece types with correct rules | |
| Entity Modeling | Piece hierarchy with move generation per subclass, Board, Move, MoveHistory | |
| Design Patterns | Command (undo/redo), polymorphism on pieces (Strategy-like) | |
| SOLID | OCP — new piece type = new class; SRP — piece knows its own moves | |
| Extensibility | New piece type = new class; special moves as MoveCommand subclasses | |
| Concurrency | Correctly identifies this is NOT concurrent (sequential game) | |
| Communication | Explained WHY Command for undo, WHY piece encapsulates move generation | |
| **TOTAL** | | **/21** |
