# LLD Interview Problems - Part 2

## Overview
This document continues with more advanced Low-Level Design interview problems including Chess Game, Hotel Booking System, and Library Management System.

---

## Chess Game

### Requirements

**Functional Requirements:**
- Standard 8x8 chess board
- All piece types with valid movement rules
- Turn-based gameplay
- Check, checkmate, and stalemate detection
- Move validation

### Class Design

```
┌─────────────────────────────────────────────────────────────────┐
│                        CHESS GAME                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐             │
│  │    Game    │───>│   Board    │───>│   Cell     │             │
│  └────────────┘    └────────────┘    └────────────┘             │
│        │                                    │                    │
│        │                                    │                    │
│        ▼                                    ▼                    │
│  ┌────────────┐                      ┌────────────┐             │
│  │   Player   │                      │   Piece    │             │
│  └────────────┘                      └────────────┘             │
│                                            △                     │
│                     ┌────────┬─────────┬───┴───┬─────────┐      │
│                     │        │         │       │         │      │
│                   King    Queen      Rook   Bishop   Knight     │
│                                                          Pawn   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Enums
public enum Color {
    WHITE, BLACK;
    
    public Color opposite() {
        return this == WHITE ? BLACK : WHITE;
    }
}

public enum PieceType {
    KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN
}

// Position
public record Position(int row, int col) {
    public boolean isValid() {
        return row >= 0 && row < 8 && col >= 0 && col < 8;
    }
    
    public Position move(int dRow, int dCol) {
        return new Position(row + dRow, col + dCol);
    }
    
    public static Position fromNotation(String notation) {
        // e.g., "e4" -> Position(4, 4)
        char colChar = notation.charAt(0);
        int row = Character.getNumericValue(notation.charAt(1)) - 1;
        int col = colChar - 'a';
        return new Position(row, col);
    }
    
    public String toNotation() {
        char col = (char) ('a' + this.col);
        return "" + col + (row + 1);
    }
}

// Move
public record Move(Position from, Position to, Piece piece, Piece captured, boolean isCheck) {
    public String toNotation() {
        String notation = piece.getSymbol() + from.toNotation() + 
            (captured != null ? "x" : "-") + to.toNotation();
        return isCheck ? notation + "+" : notation;
    }
}

// Abstract Piece
public abstract class Piece {
    protected final Color color;
    protected final PieceType type;
    protected Position position;
    protected boolean hasMoved;
    
    protected Piece(Color color, PieceType type, Position position) {
        this.color = color;
        this.type = type;
        this.position = position;
        this.hasMoved = false;
    }
    
    public abstract List<Position> getPossibleMoves(Board board);
    public abstract String getSymbol();
    
    public List<Position> getValidMoves(Board board) {
        return getPossibleMoves(board).stream()
            .filter(pos -> !board.wouldResultInCheck(this, pos))
            .toList();
    }
    
    public boolean canMoveTo(Position target, Board board) {
        return getValidMoves(board).contains(target);
    }
    
    protected List<Position> getMovesInDirection(Board board, int dRow, int dCol) {
        List<Position> moves = new ArrayList<>();
        Position current = position.move(dRow, dCol);
        
        while (current.isValid()) {
            Piece pieceAtPosition = board.getPieceAt(current);
            if (pieceAtPosition == null) {
                moves.add(current);
            } else {
                if (pieceAtPosition.getColor() != this.color) {
                    moves.add(current); // Can capture
                }
                break; // Blocked
            }
            current = current.move(dRow, dCol);
        }
        return moves;
    }
    
    // Getters
    public Color getColor() { return color; }
    public PieceType getType() { return type; }
    public Position getPosition() { return position; }
    public void setPosition(Position position) { 
        this.position = position;
        this.hasMoved = true;
    }
}

// Concrete Pieces
public class King extends Piece {
    private static final int[][] MOVES = {
        {-1, -1}, {-1, 0}, {-1, 1},
        {0, -1},           {0, 1},
        {1, -1},  {1, 0},  {1, 1}
    };
    
    public King(Color color, Position position) {
        super(color, PieceType.KING, position);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        
        for (int[] move : MOVES) {
            Position newPos = position.move(move[0], move[1]);
            if (newPos.isValid()) {
                Piece pieceAtPosition = board.getPieceAt(newPos);
                if (pieceAtPosition == null || pieceAtPosition.getColor() != this.color) {
                    moves.add(newPos);
                }
            }
        }
        
        // Castling
        if (!hasMoved && !board.isKingInCheck(color)) {
            // King-side castling
            if (canCastle(board, true)) {
                moves.add(position.move(0, 2));
            }
            // Queen-side castling
            if (canCastle(board, false)) {
                moves.add(position.move(0, -2));
            }
        }
        
        return moves;
    }
    
    private boolean canCastle(Board board, boolean kingSide) {
        int rookCol = kingSide ? 7 : 0;
        int direction = kingSide ? 1 : -1;
        
        Piece rook = board.getPieceAt(new Position(position.row(), rookCol));
        if (rook == null || rook.hasMoved || rook.getType() != PieceType.ROOK) {
            return false;
        }
        
        // Check if path is clear
        int startCol = position.col() + direction;
        int endCol = kingSide ? 6 : 1;
        for (int col = Math.min(startCol, endCol); col <= Math.max(startCol, endCol); col++) {
            if (board.getPieceAt(new Position(position.row(), col)) != null) {
                return false;
            }
        }
        
        // Check if king passes through check
        for (int i = 0; i <= 2; i++) {
            Position pos = position.move(0, direction * i);
            if (board.isPositionUnderAttack(pos, color.opposite())) {
                return false;
            }
        }
        
        return true;
    }
    
    @Override
    public String getSymbol() {
        return color == Color.WHITE ? "♔" : "♚";
    }
}

public class Queen extends Piece {
    public Queen(Color color, Position position) {
        super(color, PieceType.QUEEN, position);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        // Rook-like moves
        moves.addAll(getMovesInDirection(board, 0, 1));
        moves.addAll(getMovesInDirection(board, 0, -1));
        moves.addAll(getMovesInDirection(board, 1, 0));
        moves.addAll(getMovesInDirection(board, -1, 0));
        // Bishop-like moves
        moves.addAll(getMovesInDirection(board, 1, 1));
        moves.addAll(getMovesInDirection(board, 1, -1));
        moves.addAll(getMovesInDirection(board, -1, 1));
        moves.addAll(getMovesInDirection(board, -1, -1));
        return moves;
    }
    
    @Override
    public String getSymbol() {
        return color == Color.WHITE ? "♕" : "♛";
    }
}

public class Rook extends Piece {
    public Rook(Color color, Position position) {
        super(color, PieceType.ROOK, position);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        moves.addAll(getMovesInDirection(board, 0, 1));
        moves.addAll(getMovesInDirection(board, 0, -1));
        moves.addAll(getMovesInDirection(board, 1, 0));
        moves.addAll(getMovesInDirection(board, -1, 0));
        return moves;
    }
    
    @Override
    public String getSymbol() {
        return color == Color.WHITE ? "♖" : "♜";
    }
}

public class Bishop extends Piece {
    public Bishop(Color color, Position position) {
        super(color, PieceType.BISHOP, position);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        moves.addAll(getMovesInDirection(board, 1, 1));
        moves.addAll(getMovesInDirection(board, 1, -1));
        moves.addAll(getMovesInDirection(board, -1, 1));
        moves.addAll(getMovesInDirection(board, -1, -1));
        return moves;
    }
    
    @Override
    public String getSymbol() {
        return color == Color.WHITE ? "♗" : "♝";
    }
}

public class Knight extends Piece {
    private static final int[][] MOVES = {
        {-2, -1}, {-2, 1}, {-1, -2}, {-1, 2},
        {1, -2}, {1, 2}, {2, -1}, {2, 1}
    };
    
    public Knight(Color color, Position position) {
        super(color, PieceType.KNIGHT, position);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        for (int[] move : MOVES) {
            Position newPos = position.move(move[0], move[1]);
            if (newPos.isValid()) {
                Piece pieceAtPosition = board.getPieceAt(newPos);
                if (pieceAtPosition == null || pieceAtPosition.getColor() != this.color) {
                    moves.add(newPos);
                }
            }
        }
        return moves;
    }
    
    @Override
    public String getSymbol() {
        return color == Color.WHITE ? "♘" : "♞";
    }
}

public class Pawn extends Piece {
    public Pawn(Color color, Position position) {
        super(color, PieceType.PAWN, position);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int direction = color == Color.WHITE ? 1 : -1;
        
        // Forward move
        Position oneStep = position.move(direction, 0);
        if (oneStep.isValid() && board.getPieceAt(oneStep) == null) {
            moves.add(oneStep);
            
            // Two-step from starting position
            if (!hasMoved) {
                Position twoStep = position.move(direction * 2, 0);
                if (board.getPieceAt(twoStep) == null) {
                    moves.add(twoStep);
                }
            }
        }
        
        // Diagonal captures
        Position leftCapture = position.move(direction, -1);
        Position rightCapture = position.move(direction, 1);
        
        for (Position capture : List.of(leftCapture, rightCapture)) {
            if (capture.isValid()) {
                Piece piece = board.getPieceAt(capture);
                if (piece != null && piece.getColor() != this.color) {
                    moves.add(capture);
                }
            }
        }
        
        // En passant (simplified - would need last move tracking)
        
        return moves;
    }
    
    @Override
    public String getSymbol() {
        return color == Color.WHITE ? "♙" : "♟";
    }
}

// Board
public class Board {
    private final Piece[][] squares = new Piece[8][8];
    private final List<Piece> whitePieces = new ArrayList<>();
    private final List<Piece> blackPieces = new ArrayList<>();
    
    public Board() {
        setupInitialPosition();
    }
    
    private void setupInitialPosition() {
        // White pieces
        placePiece(new Rook(Color.WHITE, new Position(0, 0)));
        placePiece(new Knight(Color.WHITE, new Position(0, 1)));
        placePiece(new Bishop(Color.WHITE, new Position(0, 2)));
        placePiece(new Queen(Color.WHITE, new Position(0, 3)));
        placePiece(new King(Color.WHITE, new Position(0, 4)));
        placePiece(new Bishop(Color.WHITE, new Position(0, 5)));
        placePiece(new Knight(Color.WHITE, new Position(0, 6)));
        placePiece(new Rook(Color.WHITE, new Position(0, 7)));
        for (int col = 0; col < 8; col++) {
            placePiece(new Pawn(Color.WHITE, new Position(1, col)));
        }
        
        // Black pieces
        placePiece(new Rook(Color.BLACK, new Position(7, 0)));
        placePiece(new Knight(Color.BLACK, new Position(7, 1)));
        placePiece(new Bishop(Color.BLACK, new Position(7, 2)));
        placePiece(new Queen(Color.BLACK, new Position(7, 3)));
        placePiece(new King(Color.BLACK, new Position(7, 4)));
        placePiece(new Bishop(Color.BLACK, new Position(7, 5)));
        placePiece(new Knight(Color.BLACK, new Position(7, 6)));
        placePiece(new Rook(Color.BLACK, new Position(7, 7)));
        for (int col = 0; col < 8; col++) {
            placePiece(new Pawn(Color.BLACK, new Position(6, col)));
        }
    }
    
    private void placePiece(Piece piece) {
        Position pos = piece.getPosition();
        squares[pos.row()][pos.col()] = piece;
        if (piece.getColor() == Color.WHITE) {
            whitePieces.add(piece);
        } else {
            blackPieces.add(piece);
        }
    }
    
    public Piece getPieceAt(Position position) {
        if (!position.isValid()) return null;
        return squares[position.row()][position.col()];
    }
    
    public Move movePiece(Position from, Position to) {
        Piece piece = getPieceAt(from);
        Piece captured = getPieceAt(to);
        
        // Remove captured piece
        if (captured != null) {
            if (captured.getColor() == Color.WHITE) {
                whitePieces.remove(captured);
            } else {
                blackPieces.remove(captured);
            }
        }
        
        // Move piece
        squares[from.row()][from.col()] = null;
        squares[to.row()][to.col()] = piece;
        piece.setPosition(to);
        
        // Handle castling
        if (piece.getType() == PieceType.KING && Math.abs(from.col() - to.col()) == 2) {
            handleCastling(from, to);
        }
        
        // Handle pawn promotion
        if (piece.getType() == PieceType.PAWN) {
            if ((piece.getColor() == Color.WHITE && to.row() == 7) ||
                (piece.getColor() == Color.BLACK && to.row() == 0)) {
                promotePawn(piece, to);
            }
        }
        
        boolean isCheck = isKingInCheck(piece.getColor().opposite());
        return new Move(from, to, piece, captured, isCheck);
    }
    
    private void handleCastling(Position from, Position to) {
        int rookFromCol = to.col() > from.col() ? 7 : 0;
        int rookToCol = to.col() > from.col() ? to.col() - 1 : to.col() + 1;
        
        Piece rook = squares[from.row()][rookFromCol];
        squares[from.row()][rookFromCol] = null;
        squares[from.row()][rookToCol] = rook;
        rook.setPosition(new Position(from.row(), rookToCol));
    }
    
    private void promotePawn(Piece pawn, Position position) {
        // Auto-promote to Queen (could be made interactive)
        Queen queen = new Queen(pawn.getColor(), position);
        squares[position.row()][position.col()] = queen;
        
        if (pawn.getColor() == Color.WHITE) {
            whitePieces.remove(pawn);
            whitePieces.add(queen);
        } else {
            blackPieces.remove(pawn);
            blackPieces.add(queen);
        }
    }
    
    public boolean isKingInCheck(Color color) {
        Position kingPosition = findKing(color);
        return isPositionUnderAttack(kingPosition, color.opposite());
    }
    
    public boolean isPositionUnderAttack(Position position, Color attackerColor) {
        List<Piece> attackerPieces = attackerColor == Color.WHITE ? whitePieces : blackPieces;
        return attackerPieces.stream()
            .anyMatch(piece -> piece.getPossibleMoves(this).contains(position));
    }
    
    public boolean wouldResultInCheck(Piece piece, Position to) {
        // Simulate move
        Position from = piece.getPosition();
        Piece captured = getPieceAt(to);
        
        squares[from.row()][from.col()] = null;
        squares[to.row()][to.col()] = piece;
        Position originalPosition = piece.getPosition();
        piece.position = to;
        
        boolean inCheck = isKingInCheck(piece.getColor());
        
        // Undo move
        squares[from.row()][from.col()] = piece;
        squares[to.row()][to.col()] = captured;
        piece.position = originalPosition;
        
        return inCheck;
    }
    
    private Position findKing(Color color) {
        List<Piece> pieces = color == Color.WHITE ? whitePieces : blackPieces;
        return pieces.stream()
            .filter(p -> p.getType() == PieceType.KING)
            .findFirst()
            .map(Piece::getPosition)
            .orElseThrow(() -> new IllegalStateException("King not found"));
    }
    
    public boolean isCheckmate(Color color) {
        if (!isKingInCheck(color)) return false;
        return hasNoValidMoves(color);
    }
    
    public boolean isStalemate(Color color) {
        if (isKingInCheck(color)) return false;
        return hasNoValidMoves(color);
    }
    
    private boolean hasNoValidMoves(Color color) {
        List<Piece> pieces = color == Color.WHITE ? whitePieces : blackPieces;
        return pieces.stream()
            .allMatch(piece -> piece.getValidMoves(this).isEmpty());
    }
    
    public void display() {
        System.out.println("  a b c d e f g h");
        for (int row = 7; row >= 0; row--) {
            System.out.print((row + 1) + " ");
            for (int col = 0; col < 8; col++) {
                Piece piece = squares[row][col];
                System.out.print(piece != null ? piece.getSymbol() : "·");
                System.out.print(" ");
            }
            System.out.println(row + 1);
        }
        System.out.println("  a b c d e f g h");
    }
}

// Player
public class Player {
    private final String name;
    private final Color color;
    
    public Player(String name, Color color) {
        this.name = name;
        this.color = color;
    }
    
    public String getName() { return name; }
    public Color getColor() { return color; }
}

// Game
public class ChessGame {
    private final Board board;
    private final Player whitePlayer;
    private final Player blackPlayer;
    private Player currentPlayer;
    private final List<Move> moveHistory;
    private GameStatus status;
    
    public enum GameStatus {
        ACTIVE, WHITE_WINS, BLACK_WINS, STALEMATE, DRAW
    }
    
    public ChessGame(String player1Name, String player2Name) {
        this.board = new Board();
        this.whitePlayer = new Player(player1Name, Color.WHITE);
        this.blackPlayer = new Player(player2Name, Color.BLACK);
        this.currentPlayer = whitePlayer;
        this.moveHistory = new ArrayList<>();
        this.status = GameStatus.ACTIVE;
    }
    
    public MoveResult makeMove(String from, String to) {
        if (status != GameStatus.ACTIVE) {
            return new MoveResult(false, "Game is over");
        }
        
        Position fromPos = Position.fromNotation(from);
        Position toPos = Position.fromNotation(to);
        
        Piece piece = board.getPieceAt(fromPos);
        
        // Validation
        if (piece == null) {
            return new MoveResult(false, "No piece at " + from);
        }
        if (piece.getColor() != currentPlayer.getColor()) {
            return new MoveResult(false, "Not your piece");
        }
        if (!piece.canMoveTo(toPos, board)) {
            return new MoveResult(false, "Invalid move for " + piece.getType());
        }
        
        // Make move
        Move move = board.movePiece(fromPos, toPos);
        moveHistory.add(move);
        
        // Check game status
        Color opponentColor = currentPlayer.getColor().opposite();
        if (board.isCheckmate(opponentColor)) {
            status = currentPlayer.getColor() == Color.WHITE ? 
                GameStatus.WHITE_WINS : GameStatus.BLACK_WINS;
            return new MoveResult(true, "Checkmate! " + currentPlayer.getName() + " wins!");
        }
        if (board.isStalemate(opponentColor)) {
            status = GameStatus.STALEMATE;
            return new MoveResult(true, "Stalemate! Game is a draw.");
        }
        
        // Switch turn
        currentPlayer = currentPlayer == whitePlayer ? blackPlayer : whitePlayer;
        
        String message = move.toNotation();
        if (move.isCheck()) {
            message += " - Check!";
        }
        return new MoveResult(true, message);
    }
    
    public void displayBoard() {
        board.display();
        System.out.println("\n" + currentPlayer.getName() + "'s turn (" + 
            currentPlayer.getColor() + ")");
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        ChessGame game = new ChessGame("Alice", "Bob");
        game.displayBoard();
        
        // Scholar's mate
        game.makeMove("e2", "e4");
        game.displayBoard();
        
        game.makeMove("e7", "e5");
        game.displayBoard();
    }
}
```

---

## Hotel Booking System

### Requirements

**Functional Requirements:**
- Search available rooms by date, type
- Book rooms with guest details
- Cancel bookings
- Check-in/check-out
- Handle different room types and pricing

### Implementation

```java
// Enums
public enum RoomType {
    SINGLE(1, new BigDecimal("100")),
    DOUBLE(2, new BigDecimal("150")),
    SUITE(4, new BigDecimal("300")),
    PRESIDENTIAL(6, new BigDecimal("1000"));
    
    private final int capacity;
    private final BigDecimal basePrice;
    
    RoomType(int capacity, BigDecimal basePrice) {
        this.capacity = capacity;
        this.basePrice = basePrice;
    }
    
    public int getCapacity() { return capacity; }
    public BigDecimal getBasePrice() { return basePrice; }
}

public enum BookingStatus {
    CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED
}

public enum RoomStatus {
    AVAILABLE, OCCUPIED, MAINTENANCE, CLEANING
}

// Room
public class Room {
    private final String roomNumber;
    private final RoomType type;
    private final int floor;
    private RoomStatus status;
    private final Set<String> amenities;
    
    public Room(String roomNumber, RoomType type, int floor) {
        this.roomNumber = roomNumber;
        this.type = type;
        this.floor = floor;
        this.status = RoomStatus.AVAILABLE;
        this.amenities = new HashSet<>();
    }
    
    public void addAmenity(String amenity) {
        amenities.add(amenity);
    }
    
    // Getters and setters
    public String getRoomNumber() { return roomNumber; }
    public RoomType getType() { return type; }
    public int getFloor() { return floor; }
    public RoomStatus getStatus() { return status; }
    public void setStatus(RoomStatus status) { this.status = status; }
    public Set<String> getAmenities() { return Set.copyOf(amenities); }
}

// Guest
public class Guest {
    private final String id;
    private final String name;
    private final String email;
    private final String phone;
    private final String idProof;
    
    public Guest(String id, String name, String email, String phone, String idProof) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.idProof = idProof;
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}

// Booking
public class Booking {
    private final String bookingId;
    private final Guest guest;
    private final Room room;
    private final LocalDate checkInDate;
    private final LocalDate checkOutDate;
    private BookingStatus status;
    private BigDecimal totalAmount;
    private LocalDateTime actualCheckIn;
    private LocalDateTime actualCheckOut;
    
    public Booking(Guest guest, Room room, LocalDate checkInDate, LocalDate checkOutDate) {
        this.bookingId = generateBookingId();
        this.guest = guest;
        this.room = room;
        this.checkInDate = checkInDate;
        this.checkOutDate = checkOutDate;
        this.status = BookingStatus.CONFIRMED;
        this.totalAmount = calculateTotalAmount();
    }
    
    private String generateBookingId() {
        return "BK-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
    
    private BigDecimal calculateTotalAmount() {
        long nights = ChronoUnit.DAYS.between(checkInDate, checkOutDate);
        return room.getType().getBasePrice().multiply(BigDecimal.valueOf(nights));
    }
    
    public void checkIn() {
        if (status != BookingStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot check in - booking status: " + status);
        }
        this.status = BookingStatus.CHECKED_IN;
        this.actualCheckIn = LocalDateTime.now();
        room.setStatus(RoomStatus.OCCUPIED);
    }
    
    public void checkOut() {
        if (status != BookingStatus.CHECKED_IN) {
            throw new IllegalStateException("Cannot check out - not checked in");
        }
        this.status = BookingStatus.CHECKED_OUT;
        this.actualCheckOut = LocalDateTime.now();
        room.setStatus(RoomStatus.CLEANING);
    }
    
    public void cancel() {
        if (status == BookingStatus.CHECKED_IN) {
            throw new IllegalStateException("Cannot cancel - already checked in");
        }
        this.status = BookingStatus.CANCELLED;
    }
    
    // Getters
    public String getBookingId() { return bookingId; }
    public Guest getGuest() { return guest; }
    public Room getRoom() { return room; }
    public LocalDate getCheckInDate() { return checkInDate; }
    public LocalDate getCheckOutDate() { return checkOutDate; }
    public BookingStatus getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
}

// Search Criteria
public class RoomSearchCriteria {
    private LocalDate checkInDate;
    private LocalDate checkOutDate;
    private RoomType roomType;
    private int guests;
    private Set<String> requiredAmenities = new HashSet<>();
    
    public RoomSearchCriteria checkIn(LocalDate date) {
        this.checkInDate = date;
        return this;
    }
    
    public RoomSearchCriteria checkOut(LocalDate date) {
        this.checkOutDate = date;
        return this;
    }
    
    public RoomSearchCriteria roomType(RoomType type) {
        this.roomType = type;
        return this;
    }
    
    public RoomSearchCriteria guests(int count) {
        this.guests = count;
        return this;
    }
    
    public RoomSearchCriteria withAmenity(String amenity) {
        this.requiredAmenities.add(amenity);
        return this;
    }
    
    // Getters
    public LocalDate getCheckInDate() { return checkInDate; }
    public LocalDate getCheckOutDate() { return checkOutDate; }
    public RoomType getRoomType() { return roomType; }
    public int getGuests() { return guests; }
    public Set<String> getRequiredAmenities() { return requiredAmenities; }
}

// Hotel Service
public class HotelService {
    private final Map<String, Room> rooms;
    private final Map<String, Booking> bookings;
    private final Map<String, List<Booking>> roomBookings; // roomNumber -> bookings
    
    public HotelService() {
        this.rooms = new ConcurrentHashMap<>();
        this.bookings = new ConcurrentHashMap<>();
        this.roomBookings = new ConcurrentHashMap<>();
    }
    
    public void addRoom(Room room) {
        rooms.put(room.getRoomNumber(), room);
        roomBookings.put(room.getRoomNumber(), new CopyOnWriteArrayList<>());
    }
    
    public List<Room> searchAvailableRooms(RoomSearchCriteria criteria) {
        return rooms.values().stream()
            .filter(room -> matchesCriteria(room, criteria))
            .filter(room -> isRoomAvailable(room, criteria.getCheckInDate(), 
                criteria.getCheckOutDate()))
            .sorted(Comparator.comparing(r -> r.getType().getBasePrice()))
            .toList();
    }
    
    private boolean matchesCriteria(Room room, RoomSearchCriteria criteria) {
        // Check room type
        if (criteria.getRoomType() != null && room.getType() != criteria.getRoomType()) {
            return false;
        }
        
        // Check capacity
        if (criteria.getGuests() > 0 && room.getType().getCapacity() < criteria.getGuests()) {
            return false;
        }
        
        // Check amenities
        if (!criteria.getRequiredAmenities().isEmpty() && 
            !room.getAmenities().containsAll(criteria.getRequiredAmenities())) {
            return false;
        }
        
        // Check room status
        if (room.getStatus() == RoomStatus.MAINTENANCE) {
            return false;
        }
        
        return true;
    }
    
    private boolean isRoomAvailable(Room room, LocalDate checkIn, LocalDate checkOut) {
        List<Booking> bookingList = roomBookings.get(room.getRoomNumber());
        if (bookingList == null) return true;
        
        return bookingList.stream()
            .filter(b -> b.getStatus() == BookingStatus.CONFIRMED || 
                        b.getStatus() == BookingStatus.CHECKED_IN)
            .noneMatch(b -> datesOverlap(checkIn, checkOut, 
                b.getCheckInDate(), b.getCheckOutDate()));
    }
    
    private boolean datesOverlap(LocalDate start1, LocalDate end1, 
                                 LocalDate start2, LocalDate end2) {
        return !start1.isAfter(end2.minusDays(1)) && !end1.minusDays(1).isBefore(start2);
    }
    
    public synchronized Booking createBooking(Guest guest, String roomNumber, 
                                              LocalDate checkIn, LocalDate checkOut) {
        Room room = rooms.get(roomNumber);
        if (room == null) {
            throw new IllegalArgumentException("Room not found: " + roomNumber);
        }
        
        if (!isRoomAvailable(room, checkIn, checkOut)) {
            throw new IllegalStateException("Room not available for selected dates");
        }
        
        Booking booking = new Booking(guest, room, checkIn, checkOut);
        bookings.put(booking.getBookingId(), booking);
        roomBookings.get(roomNumber).add(booking);
        
        return booking;
    }
    
    public void cancelBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found: " + bookingId);
        }
        booking.cancel();
    }
    
    public void checkIn(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found: " + bookingId);
        }
        booking.checkIn();
    }
    
    public Invoice checkOut(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found: " + bookingId);
        }
        booking.checkOut();
        
        // Generate invoice
        return new Invoice(booking);
    }
    
    public Booking getBooking(String bookingId) {
        return bookings.get(bookingId);
    }
}

// Invoice
public class Invoice {
    private final String invoiceId;
    private final Booking booking;
    private final LocalDateTime generatedAt;
    private final BigDecimal roomCharges;
    private final BigDecimal taxes;
    private final BigDecimal totalAmount;
    
    public Invoice(Booking booking) {
        this.invoiceId = "INV-" + System.currentTimeMillis();
        this.booking = booking;
        this.generatedAt = LocalDateTime.now();
        this.roomCharges = booking.getTotalAmount();
        this.taxes = roomCharges.multiply(new BigDecimal("0.18")); // 18% tax
        this.totalAmount = roomCharges.add(taxes);
    }
    
    public void print() {
        System.out.println("╔══════════════════════════════════════╗");
        System.out.println("║           HOTEL INVOICE              ║");
        System.out.println("╠══════════════════════════════════════╣");
        System.out.printf("║ Invoice #: %-26s ║%n", invoiceId);
        System.out.printf("║ Booking #: %-26s ║%n", booking.getBookingId());
        System.out.printf("║ Guest: %-30s ║%n", booking.getGuest().getName());
        System.out.printf("║ Room: %-31s ║%n", booking.getRoom().getRoomNumber());
        System.out.printf("║ Check-in: %-27s ║%n", booking.getCheckInDate());
        System.out.printf("║ Check-out: %-26s ║%n", booking.getCheckOutDate());
        System.out.println("╠══════════════════════════════════════╣");
        System.out.printf("║ Room Charges: %23s ║%n", "$" + roomCharges);
        System.out.printf("║ Taxes (18%%): %24s ║%n", "$" + taxes);
        System.out.println("╠══════════════════════════════════════╣");
        System.out.printf("║ TOTAL: %30s ║%n", "$" + totalAmount);
        System.out.println("╚══════════════════════════════════════╝");
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        HotelService hotel = new HotelService();
        
        // Add rooms
        Room r101 = new Room("101", RoomType.SINGLE, 1);
        r101.addAmenity("WiFi");
        r101.addAmenity("TV");
        hotel.addRoom(r101);
        
        Room r201 = new Room("201", RoomType.DOUBLE, 2);
        r201.addAmenity("WiFi");
        r201.addAmenity("TV");
        r201.addAmenity("Mini Bar");
        hotel.addRoom(r201);
        
        Room r301 = new Room("301", RoomType.SUITE, 3);
        r301.addAmenity("WiFi");
        r301.addAmenity("TV");
        r301.addAmenity("Mini Bar");
        r301.addAmenity("Jacuzzi");
        hotel.addRoom(r301);
        
        // Search for rooms
        RoomSearchCriteria criteria = new RoomSearchCriteria()
            .checkIn(LocalDate.now().plusDays(1))
            .checkOut(LocalDate.now().plusDays(3))
            .guests(2)
            .withAmenity("WiFi");
        
        List<Room> available = hotel.searchAvailableRooms(criteria);
        System.out.println("Available rooms: " + available.size());
        
        // Create booking
        Guest guest = new Guest("G001", "John Doe", "john@email.com", 
            "+1234567890", "DL12345");
        Booking booking = hotel.createBooking(guest, "201", 
            LocalDate.now().plusDays(1), LocalDate.now().plusDays(3));
        System.out.println("Booking created: " + booking.getBookingId());
        
        // Check-in
        hotel.checkIn(booking.getBookingId());
        System.out.println("Checked in!");
        
        // Check-out
        Invoice invoice = hotel.checkOut(booking.getBookingId());
        invoice.print();
    }
}
```

---

## Library Management System

### Requirements

**Functional Requirements:**
- Add/remove books
- Register members
- Issue and return books
- Search books by title/author/ISBN
- Track due dates and fines

### Implementation

```java
// Book
public class Book {
    private final String isbn;
    private final String title;
    private final String author;
    private final String publisher;
    private final int publicationYear;
    private final String category;
    
    public Book(String isbn, String title, String author, 
                String publisher, int publicationYear, String category) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
        this.publisher = publisher;
        this.publicationYear = publicationYear;
        this.category = category;
    }
    
    // Getters
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public String getCategory() { return category; }
}

// Book Copy
public class BookCopy {
    private final String copyId;
    private final Book book;
    private boolean available;
    private String rackNumber;
    
    public BookCopy(String copyId, Book book, String rackNumber) {
        this.copyId = copyId;
        this.book = book;
        this.rackNumber = rackNumber;
        this.available = true;
    }
    
    public void markBorrowed() { this.available = false; }
    public void markReturned() { this.available = true; }
    
    // Getters
    public String getCopyId() { return copyId; }
    public Book getBook() { return book; }
    public boolean isAvailable() { return available; }
    public String getRackNumber() { return rackNumber; }
}

// Member
public class Member {
    private final String memberId;
    private final String name;
    private final String email;
    private final LocalDate memberSince;
    private final int maxBooks;
    private BigDecimal fineAmount;
    
    public Member(String memberId, String name, String email, int maxBooks) {
        this.memberId = memberId;
        this.name = name;
        this.email = email;
        this.memberSince = LocalDate.now();
        this.maxBooks = maxBooks;
        this.fineAmount = BigDecimal.ZERO;
    }
    
    public void addFine(BigDecimal amount) {
        this.fineAmount = this.fineAmount.add(amount);
    }
    
    public void payFine(BigDecimal amount) {
        this.fineAmount = this.fineAmount.subtract(amount);
        if (this.fineAmount.compareTo(BigDecimal.ZERO) < 0) {
            this.fineAmount = BigDecimal.ZERO;
        }
    }
    
    // Getters
    public String getMemberId() { return memberId; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public int getMaxBooks() { return maxBooks; }
    public BigDecimal getFineAmount() { return fineAmount; }
}

// Loan Record
public class LoanRecord {
    private final String loanId;
    private final BookCopy bookCopy;
    private final Member member;
    private final LocalDate issueDate;
    private final LocalDate dueDate;
    private LocalDate returnDate;
    private BigDecimal fineAmount;
    
    private static final int LOAN_PERIOD_DAYS = 14;
    private static final BigDecimal FINE_PER_DAY = new BigDecimal("0.50");
    
    public LoanRecord(BookCopy bookCopy, Member member) {
        this.loanId = "LN-" + UUID.randomUUID().toString().substring(0, 8);
        this.bookCopy = bookCopy;
        this.member = member;
        this.issueDate = LocalDate.now();
        this.dueDate = issueDate.plusDays(LOAN_PERIOD_DAYS);
        this.fineAmount = BigDecimal.ZERO;
    }
    
    public void markReturned() {
        this.returnDate = LocalDate.now();
        calculateFine();
    }
    
    private void calculateFine() {
        if (returnDate.isAfter(dueDate)) {
            long daysLate = ChronoUnit.DAYS.between(dueDate, returnDate);
            this.fineAmount = FINE_PER_DAY.multiply(BigDecimal.valueOf(daysLate));
            member.addFine(this.fineAmount);
        }
    }
    
    public boolean isOverdue() {
        if (returnDate != null) return false;
        return LocalDate.now().isAfter(dueDate);
    }
    
    // Getters
    public String getLoanId() { return loanId; }
    public BookCopy getBookCopy() { return bookCopy; }
    public Member getMember() { return member; }
    public LocalDate getDueDate() { return dueDate; }
    public LocalDate getReturnDate() { return returnDate; }
    public BigDecimal getFineAmount() { return fineAmount; }
}

// Search Strategy
public interface BookSearchStrategy {
    List<Book> search(Collection<Book> books, String query);
}

public class TitleSearchStrategy implements BookSearchStrategy {
    @Override
    public List<Book> search(Collection<Book> books, String query) {
        String lowerQuery = query.toLowerCase();
        return books.stream()
            .filter(b -> b.getTitle().toLowerCase().contains(lowerQuery))
            .toList();
    }
}

public class AuthorSearchStrategy implements BookSearchStrategy {
    @Override
    public List<Book> search(Collection<Book> books, String query) {
        String lowerQuery = query.toLowerCase();
        return books.stream()
            .filter(b -> b.getAuthor().toLowerCase().contains(lowerQuery))
            .toList();
    }
}

public class IsbnSearchStrategy implements BookSearchStrategy {
    @Override
    public List<Book> search(Collection<Book> books, String query) {
        return books.stream()
            .filter(b -> b.getIsbn().equals(query))
            .toList();
    }
}

// Library Service
public class LibraryService {
    private final Map<String, Book> books; // ISBN -> Book
    private final Map<String, List<BookCopy>> bookCopies; // ISBN -> copies
    private final Map<String, Member> members;
    private final Map<String, LoanRecord> activeLoans; // copyId -> loan
    private final List<LoanRecord> loanHistory;
    
    public LibraryService() {
        this.books = new ConcurrentHashMap<>();
        this.bookCopies = new ConcurrentHashMap<>();
        this.members = new ConcurrentHashMap<>();
        this.activeLoans = new ConcurrentHashMap<>();
        this.loanHistory = new CopyOnWriteArrayList<>();
    }
    
    // Book management
    public void addBook(Book book) {
        books.put(book.getIsbn(), book);
        bookCopies.putIfAbsent(book.getIsbn(), new CopyOnWriteArrayList<>());
    }
    
    public BookCopy addBookCopy(String isbn, String rackNumber) {
        Book book = books.get(isbn);
        if (book == null) {
            throw new IllegalArgumentException("Book not found: " + isbn);
        }
        
        String copyId = isbn + "-" + (bookCopies.get(isbn).size() + 1);
        BookCopy copy = new BookCopy(copyId, book, rackNumber);
        bookCopies.get(isbn).add(copy);
        return copy;
    }
    
    // Member management
    public void registerMember(Member member) {
        members.put(member.getMemberId(), member);
    }
    
    // Search
    public List<Book> searchBooks(BookSearchStrategy strategy, String query) {
        return strategy.search(books.values(), query);
    }
    
    public List<BookCopy> getAvailableCopies(String isbn) {
        List<BookCopy> copies = bookCopies.get(isbn);
        if (copies == null) return List.of();
        return copies.stream()
            .filter(BookCopy::isAvailable)
            .toList();
    }
    
    // Loan operations
    public synchronized LoanRecord issueBook(String memberId, String isbn) {
        Member member = members.get(memberId);
        if (member == null) {
            throw new IllegalArgumentException("Member not found: " + memberId);
        }
        
        // Check member's fine
        if (member.getFineAmount().compareTo(BigDecimal.ZERO) > 0) {
            throw new IllegalStateException("Member has outstanding fines: $" + 
                member.getFineAmount());
        }
        
        // Check current loans
        long currentLoans = activeLoans.values().stream()
            .filter(l -> l.getMember().getMemberId().equals(memberId))
            .count();
        if (currentLoans >= member.getMaxBooks()) {
            throw new IllegalStateException("Member has reached max books limit");
        }
        
        // Find available copy
        BookCopy copy = getAvailableCopies(isbn).stream()
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("No available copies"));
        
        // Create loan
        copy.markBorrowed();
        LoanRecord loan = new LoanRecord(copy, member);
        activeLoans.put(copy.getCopyId(), loan);
        loanHistory.add(loan);
        
        return loan;
    }
    
    public LoanRecord returnBook(String copyId) {
        LoanRecord loan = activeLoans.remove(copyId);
        if (loan == null) {
            throw new IllegalArgumentException("No active loan for copy: " + copyId);
        }
        
        loan.markReturned();
        loan.getBookCopy().markReturned();
        
        return loan;
    }
    
    public List<LoanRecord> getOverdueLoans() {
        return activeLoans.values().stream()
            .filter(LoanRecord::isOverdue)
            .toList();
    }
    
    public List<LoanRecord> getMemberLoans(String memberId) {
        return activeLoans.values().stream()
            .filter(l -> l.getMember().getMemberId().equals(memberId))
            .toList();
    }
    
    // Reservation (optional feature)
    private final Map<String, Queue<String>> reservations = new ConcurrentHashMap<>();
    
    public void reserveBook(String memberId, String isbn) {
        reservations.computeIfAbsent(isbn, k -> new ConcurrentLinkedQueue<>())
            .offer(memberId);
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        LibraryService library = new LibraryService();
        
        // Add books
        Book book1 = new Book("978-0-13-468599-1", "Clean Code", 
            "Robert Martin", "Prentice Hall", 2008, "Programming");
        library.addBook(book1);
        library.addBookCopy("978-0-13-468599-1", "A-1-1");
        library.addBookCopy("978-0-13-468599-1", "A-1-2");
        
        Book book2 = new Book("978-0-596-51774-8", "Head First Design Patterns",
            "Eric Freeman", "O'Reilly", 2004, "Programming");
        library.addBook(book2);
        library.addBookCopy("978-0-596-51774-8", "A-2-1");
        
        // Register member
        Member member = new Member("M001", "Alice Smith", "alice@email.com", 5);
        library.registerMember(member);
        
        // Search books
        List<Book> found = library.searchBooks(new AuthorSearchStrategy(), "Martin");
        System.out.println("Found " + found.size() + " books by Martin");
        
        // Issue book
        LoanRecord loan = library.issueBook("M001", "978-0-13-468599-1");
        System.out.println("Issued book. Due date: " + loan.getDueDate());
        
        // Return book
        LoanRecord returnedLoan = library.returnBook(loan.getBookCopy().getCopyId());
        System.out.println("Book returned. Fine: $" + returnedLoan.getFineAmount());
    }
}
```

---

## Quick Reference

| System | Key Patterns Used |
|--------|------------------|
| **Parking Lot** | Singleton, Strategy, Factory |
| **Elevator** | State, Strategy, Observer |
| **Vending Machine** | State |
| **Chess** | Strategy, Template Method |
| **Hotel Booking** | Builder, Strategy |
| **Library** | Strategy, Observer |

---

## Related Topics
- [[05-lld-interview-problems-1]] - Parking Lot, Elevator, Vending Machine
- [[01-lld-fundamentals]] - SOLID principles
- [[04-behavioral-patterns]] - State, Strategy patterns
