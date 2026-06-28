# Lessons Learned — Chess Game

---

## Mistake 1: Switch on Piece Type for Move Validation

**What happens:**
```java
boolean isValidMove(Piece piece, Cell from, Cell to, Board board) {
    return switch (piece.getType()) {
        case ROOK -> isValidRookMove(from, to, board);
        case KNIGHT -> isValidKnightMove(from, to);
        case BISHOP -> isValidBishopMove(from, to, board);
        // Adding a new piece = edit this method
    };
}
```

**Why it's wrong:** Adding a new piece type requires editing this method. OCP violation.

**Fix:** Each piece knows its own movement:
```java
abstract class Piece {
    public abstract List<Cell> getPossibleMoves(Board board);
}
class Rook extends Piece {
    public List<Cell> getPossibleMoves(Board board) { /* rook logic */ }
}
// New piece = new class, zero changes to Board or Game
```

**Gap dimension:** Design Patterns + SOLID (Dimensions 3, 4)

---

## Mistake 2: Board Copy for Undo (O(64) Per Move)

**What happens:** Undo implemented by saving a deep copy of the board before each move:
```java
Board savedBoard = board.deepCopy(); // copies all 64 cells
history.push(savedBoard);
// Undo: board = history.pop()
```

**Why it's wrong:** O(64) copy per move for undo. With move history of 100 moves: 6400 cell copies stored. Command pattern only stores what changed (2 cells).

**Fix:** Command pattern — store only the move and captured piece:
```java
class MoveCommand {
    private final Move move;
    private Piece capturedPiece; // set during execute()
    public void execute(Board b) { capturedPiece = move.getTo().getPiece(); /* apply move */ }
    public void undo(Board b) { /* reverse: restore from and to cells */ }
}
```

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 3: Not Validating That Move Doesn't Leave Own King in Check

**What happens:** Candidate validates piece movement rules (rook moves in straight lines) but doesn't check if the move exposes own king to attack.

**Why it's wrong:** A legal-looking rook move that reveals the king to a bishop is still illegal in chess (you can't move into check). Without this check, the game allows illegal moves.

**Fix:** After validating piece movement, simulate the move, check if own king is in check, undo:
```java
boolean isLegalMove(Board board, Move move, Player player) {
    if (!piece.getPossibleMoves(board).contains(move.getTo())) return false;
    // Simulate
    MoveCommand cmd = new MoveCommand(move);
    cmd.execute(board);
    boolean selfInCheck = isKingInCheck(board, player.getColor());
    cmd.undo(board);
    return !selfInCheck;
}
```

**Gap dimension:** Entity Modeling / Correctness (Dimension 2)

---

## Mistake 4: Storing capturedPiece in Move (Not MoveCommand)

**What happens:** `Move` object stores the captured piece. But Move is created BEFORE execution — we don't know what's captured until we execute.

**Why it's wrong:** Move is created from user input (from/to cells). `capturedPiece` is only known at execution time. Storing it in Move requires two-phase construction.

**Fix:** Store `capturedPiece` in `MoveCommand`, set during `execute()`:
```java
class MoveCommand {
    private Piece capturedPiece; // null until execute() is called
    public void execute(Board b) {
        this.capturedPiece = move.getTo().getPiece(); // capture what's there
        // then move piece
    }
    public void undo(Board b) {
        move.getTo().setPiece(capturedPiece); // restore captured piece (or null)
        // move piece back
    }
}
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 5: Treating Chess as a Concurrency Problem

**What happens:** Candidate starts adding `synchronized` keywords, thinking about concurrent moves.

**Why it's wrong:** Chess is inherently sequential — only one player moves at a time. For two-player local game: no threads, no synchronization needed. Mention this explicitly.

**Good answer:**
> "This is a sequential game — no concurrency needed for two-player local play. If we added online multiplayer, moves would be serialized server-side, and we'd only need to worry about network timeouts."

**Gap dimension:** Concurrency (Dimension 6) — correctly identifying the ABSENCE of concurrency concern is a positive signal

---

## Problem-Specific Gotchas

- Checkmate = in check AND every legal move still leaves king in check — must check ALL pieces
- Stalemate = NOT in check BUT no legal move available — don't confuse with checkmate
- `Board.findKing(color)` needed for check detection — scanning board is O(64) which is fine
- Pawn moves are the most complex: moves forward, captures diagonally, en passant, promotion
- `MoveHistory` uses TWO stacks: undoStack and redoStack — new move clears redoStack

---

## Self-Assessment Checklist

- [ ] Does each Piece subclass implement `getPossibleMoves()` (not a switch in Board)?
- [ ] Does `MoveCommand` have both `execute()` and `undo()` with `capturedPiece` stored?
- [ ] Does move validation simulate the move and check for self-check?
- [ ] Is `capturedPiece` stored in MoveCommand (not Move)?
- [ ] Does checkmate detection check ALL possible moves for the player?
- [ ] Did I explicitly note that chess is sequential (no concurrency)?
- [ ] Can a new piece type be added by adding one class only?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Design Patterns | Switch on piece type, board copy for undo | GAP_REMEDIATION.md#gap-3 — Command trigger: "need undo/redo" |
| SOLID | OCP violation on piece type | GAP_REMEDIATION.md#gap-4 — violation hunt |
| Entity Modeling | capturedPiece in wrong place, no simulate-check-undo | GAP_REMEDIATION.md#gap-2 |
| Concurrency | Added unnecessary synchronization | GAP_REMEDIATION.md#gap-6 — identify WHERE concurrency is actually needed |
