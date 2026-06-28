# Solution Guide — Chess Game

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `Game` | Concrete | Turn management, game state, move history, win detection |
| `Board` | Concrete | 8×8 grid of `Cell`; current piece positions; check detection helper |
| `Cell` | Concrete | (row, col) coordinate; holds one `Piece` or null |
| `Piece` | Abstract | Color, current position; `getPossibleMoves(board): List<Cell>` |
| `King, Queen, Rook, Bishop, Knight, Pawn` | Concrete | Each implements move generation for its movement rule |
| `Move` | Concrete | fromCell, toCell, piece, capturedPiece (null if empty square) |
| `MoveCommand` | Concrete | Wraps `Move`; `execute(board)` and `undo(board)` — Command pattern |
| `MoveHistory` | Concrete | `Deque<MoveCommand>`; `push()`, `pop()` for undo, second stack for redo |
| `MoveValidator` | Concrete | `isLegalMove(board, move, player)` — checks piece rules + doesn't leave king in check |
| `Player` | Concrete | Name, color (WHITE/BLACK) |
| `GameState` | Enum | ACTIVE, WHITE_IN_CHECK, BLACK_IN_CHECK, CHECKMATE, STALEMATE |

---

## Design Pattern 1: Polymorphism on Piece Move Generation

Each piece knows its own movement rules. No switch statement on piece type.

```java
abstract class Piece {
    protected final Color color;
    protected Cell position;

    public abstract List<Cell> getPossibleMoves(Board board);
    // Returns all GEOMETRICALLY valid cells — does NOT filter for check
    // MoveValidator filters out moves that leave own king in check

    public Color getColor() { return color; }
    public Cell getPosition() { return position; }
    public void setPosition(Cell cell) { this.position = cell; }
}

class Rook extends Piece {
    public List<Cell> getPossibleMoves(Board board) {
        List<Cell> moves = new ArrayList<>();
        // All cells in same row (left + right)
        addStraightMoves(board, moves, 0, 1);  // right
        addStraightMoves(board, moves, 0, -1); // left
        addStraightMoves(board, moves, 1, 0);  // down
        addStraightMoves(board, moves, -1, 0); // up
        return moves;
    }

    private void addStraightMoves(Board board, List<Cell> moves, int dRow, int dCol) {
        int row = position.getRow() + dRow;
        int col = position.getCol() + dCol;
        while (board.isValidCell(row, col)) {
            Cell cell = board.getCell(row, col);
            if (cell.isEmpty()) {
                moves.add(cell);
            } else {
                if (cell.getPiece().getColor() != this.color) {
                    moves.add(cell); // can capture opponent
                }
                break; // blocked by any piece
            }
            row += dRow;
            col += dCol;
        }
    }
}

class Knight extends Piece {
    private static final int[][] OFFSETS = {{-2,-1},{-2,1},{-1,-2},{-1,2},{1,-2},{1,2},{2,-1},{2,1}};

    public List<Cell> getPossibleMoves(Board board) {
        return Arrays.stream(OFFSETS)
            .map(o -> new int[]{position.getRow() + o[0], position.getCol() + o[1]})
            .filter(pos -> board.isValidCell(pos[0], pos[1]))
            .map(pos -> board.getCell(pos[0], pos[1]))
            .filter(cell -> cell.isEmpty() || cell.getPiece().getColor() != this.color)
            .collect(Collectors.toList());
    }
}
```

Adding new piece type: one new class. Zero changes to Board, Game, or any other piece.

---

## Design Pattern 2: Command — Move with Undo

```java
class MoveCommand {
    private final Move move;
    private Piece capturedPiece; // stored for undo

    public void execute(Board board) {
        Cell from = move.getFrom();
        Cell to = move.getTo();
        capturedPiece = to.getPiece(); // remember what was there (null if empty)
        to.setPiece(from.getPiece());
        from.setPiece(null);
        move.getPiece().setPosition(to);
    }

    public void undo(Board board) {
        Cell from = move.getFrom();
        Cell to = move.getTo();
        from.setPiece(to.getPiece()); // move piece back
        to.setPiece(capturedPiece);   // restore captured piece (or null)
        move.getPiece().setPosition(from);
    }
}

class MoveHistory {
    private final Deque<MoveCommand> undoStack = new ArrayDeque<>();
    private final Deque<MoveCommand> redoStack = new ArrayDeque<>();

    public void push(MoveCommand cmd) {
        undoStack.push(cmd);
        redoStack.clear(); // new move invalidates redo history
    }

    public Optional<MoveCommand> undo() {
        if (undoStack.isEmpty()) return Optional.empty();
        MoveCommand cmd = undoStack.pop();
        redoStack.push(cmd);
        return Optional.of(cmd);
    }

    public Optional<MoveCommand> redo() {
        if (redoStack.isEmpty()) return Optional.empty();
        MoveCommand cmd = redoStack.pop();
        undoStack.push(cmd);
        return Optional.of(cmd);
    }
}
```

---

## Move Validation and Check Detection

```java
class MoveValidator {
    public boolean isLegalMove(Board board, Move move, Player currentPlayer) {
        // 1. Piece belongs to current player
        if (move.getPiece().getColor() != currentPlayer.getColor()) return false;

        // 2. Destination is geometrically reachable by this piece type
        if (!move.getPiece().getPossibleMoves(board).contains(move.getTo())) return false;

        // 3. Move doesn't leave own king in check
        // Simulate the move, check for check, undo
        MoveCommand cmd = new MoveCommand(move);
        cmd.execute(board);
        boolean leavesKingInCheck = isInCheck(board, currentPlayer.getColor());
        cmd.undo(board);

        return !leavesKingInCheck;
    }

    public boolean isInCheck(Board board, Color color) {
        Cell kingCell = board.findKing(color);
        Color opponentColor = color == Color.WHITE ? Color.BLACK : Color.WHITE;
        return board.getAllPieces(opponentColor).stream()
            .anyMatch(piece -> piece.getPossibleMoves(board).contains(kingCell));
    }

    public boolean isCheckmate(Board board, Player player) {
        // Checkmate = in check AND no legal move escapes it
        if (!isInCheck(board, player.getColor())) return false;
        return board.getAllPieces(player.getColor()).stream()
            .allMatch(piece ->
                piece.getPossibleMoves(board).stream()
                    .map(to -> new Move(piece.getPosition(), to, piece))
                    .noneMatch(move -> isLegalMove(board, move, player))
            );
    }
}
```

---

## Game Loop

```java
class Game {
    private final Board board;
    private final Player[] players = new Player[2];
    private int currentPlayerIndex = 0; // 0 = White, 1 = Black
    private final MoveHistory history = new MoveHistory();
    private final MoveValidator validator = new MoveValidator();
    private GameState state = GameState.ACTIVE;

    public boolean makeMove(Player player, Cell from, Cell to) {
        if (!player.equals(players[currentPlayerIndex])) throw new NotYourTurnException();

        Move move = new Move(from, to, from.getPiece());
        if (!validator.isLegalMove(board, move, player)) return false;

        MoveCommand cmd = new MoveCommand(move);
        cmd.execute(board);
        history.push(cmd);

        // Switch turns
        currentPlayerIndex = 1 - currentPlayerIndex;
        Player nextPlayer = players[currentPlayerIndex];

        // Update game state
        if (validator.isCheckmate(board, nextPlayer)) {
            state = GameState.CHECKMATE;
        } else if (validator.isInCheck(board, nextPlayer.getColor())) {
            state = nextPlayer.getColor() == Color.WHITE ? GameState.WHITE_IN_CHECK : GameState.BLACK_IN_CHECK;
        } else {
            state = GameState.ACTIVE;
        }
        return true;
    }

    public boolean undo() {
        return history.undo().map(cmd -> { cmd.undo(board); return true; }).orElse(false);
    }
}
```

---

## Concurrency Note

Chess is inherently sequential (one move at a time). No concurrency needed for single-machine game. If online multiplayer: moves are serialized by a game server; clients submit moves to a queue.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| New piece type | New class extending `Piece`; implements `getPossibleMoves()` |
| Castling | `CastlingMove extends Move`; MoveCommand handles both king and rook |
| En passant | `EnPassantMove extends Move`; captures pawn on adjacent square |
| Pawn promotion | Detected when pawn reaches last rank; swap piece in MoveCommand.execute() |
| PGN export | `toPGN()` on each `MoveCommand`; history exports the list |
| Chess clock | Timer per player; started/stopped on `makeMove()` |

---

## What Strong Candidates Do Differently

- `getPossibleMoves()` on each Piece subclass — not a switch in Board or Game
- Command pattern for undo immediately (not "copy the whole board state")
- Check detection via simulate-execute-check-undo — elegant, correct
- `capturedPiece` stored in `MoveCommand` for undo restoration
- Explicitly notes: Chess is sequential — no concurrency concern here

## What Average Candidates Miss

- Switch on piece type in one giant `validateMove()` method
- Board copy for undo — O(64) deep copy each move, wasteful
- Check detection only for capture moves (misses non-capture moves that expose king)
- No undo mechanism at all
- Piece knows about the game (not just the board) — wrong coupling level
