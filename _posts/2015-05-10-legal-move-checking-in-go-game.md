---
layout: post
title: Legal Move Checking in Go Game
description:
categories:
- Scala
tags:
- scala
- go game
published: true
---

Last month I attended two coding dojo sessions in [Singapore Scala Programmers](http://www.meetup.com/Singapore-Scala-Programmers) group. We had a kata for writing a Go Game program that can determine whether a move is legal. In this post I will share with you my implementation of the Go's board as well as the move checking. To understand about Go Game as well as its rule, you can read it on [Wikipedia](http://en.wikipedia.org/wiki/Go_(game)) or the kata's description on [GitHub](https://github.com/luugiathuy/gokata).

## Piece

```scala
sealed abstract class Piece {
  val opponentPiece: Piece = Empty
}

case object Empty extends Piece

case object BlackPiece extends Piece {
  override val opponentPiece = WhitePiece
}

case object WhitePiece extends Piece {
  override val opponentPiece = BlackPiece
}
```

I defined an abstract `Piece` class with 3 case objects: `Empty` (for representing unoccupied positions), `BlackPiece` and `WhitePiece`. These classes all have `opponentPiece` function, but mostly used by BlackPiece and WhitePiece to get each other piece.

## Move

```scala
case class Move(x: Int, y: Int, piece: Piece) {
  require(piece != Empty, "Invalid Piece, only BlackPiece or WhitePiece")
}
```

The `Move` class's constructor has 3 paramters, `x` and `y` represent the row and column coordinate on the board respectively, `piece` represents which player plays the move. Note that `piece` cannot be an `Empty` type.

## GoGame

With `Piece` and `Move` classes above, I wrote a `GoGameDef` trait which defines logics of a go game:

```scala
trait GoGameDef {

  val rowCount: Int
  val colCount: Int

  type Positions = Vector[Vector[Piece]]

  case class Board(positions: Positions = Vector.fill(rowCount, colCount)(Empty),
                   nextPiece: Piece = BlackPiece) {
    require(positions.length == rowCount && positions.head.length == colCount,
      "Invalid positions length")
  }

  /**
   * A record of boards of the game.
   * The Stream.head is the current board
   */
  type BoardRecord = Stream[Board]

  def isLegalMove(move: Move, boardRecord: BoardRecord): Boolean = ???

  /**
   * Given a move and board record, play a move on the board, capture no-liberty opponent's pieces
   * If a move is illegal, an IllegalArgumentException exception will be thrown
   * @return a new board record
   */
  def playMove(move: Move, boardRecord: BoardRecord): BoardRecord = ???
```

`rowCount` and `colCount` are for defining the size of the board. A standard Go board has the size of 19&times;19. 13&times;13 and 9&times;9 boards are popular choices to teach beginners.

The `Positions` type is a 2D Vector of `Piece` objects which represents the state of all positions of a board, whether it is unoccupied `Empty`, occupied by `BlackPiece` or `WhitePiece`. Next we have a `Board` class which has a `positions` to hold the state of the board and `nextPiece` to indicate which player is going to play next. I define a `BoardRecord` type which is a `Stream` of `Board` objects to record all `Board` states of a Go Game after each move.

Given a `Move` and a `BoardRecord`, the `isLegalMove` method returns `true` if the move is legal, false otherwise. I also have `playMove` method which simulate playing a move and returns a new `BoardRecord`. Our kata's objective is implementing the `isLegalMove` method.

## Test Cases

Before implementing the `isLegalMove` method, I wrote ScalaTest specs so that the method needs to check for all rules of a move. To define a board game easily, I wrote a `StringParserGoGame` which helps to parse a `Board` object from ASCII string:

```scala
/**
 * This trait implements a parser to define a board state from a ASCII string.
 *
 * When mixing in this trait, we can get the `initialBoardRecord`
 * by defining the field `boardASCII` in the following form:
 *
 *   val boardASCII =
 *     """x
 *       |-----
 *       |--o--
 *       |--x--
 *       |-----""".stripMargin
 *
 * - The first line will have one character `x` or `o` denotes
 *    the next piece
 * - The following lines represent the board's position
 * - The `-` character denotes empty positions
 * - `x` denotes positions which are occupied by black
 * - `o` denotes positions which are occupied by white
 *
 */
trait StringParserGoGame extends GoGameDef {

  val boardASCII: String

  lazy val rowCount = positions.length
  lazy val colCount = positions.head.length

  private lazy val positions: Positions = {
    parseBoardPositions(boardASCII)
  }

  protected lazy val initialBoardRecord: BoardRecord = {
    val newBoard = {
      val nextPiece = if (boardASCII.charAt(0) == 'o') WhitePiece else BlackPiece
      Board(positions, nextPiece)
    }
    Stream(newBoard)
  }

  protected def parseBoardPositions(boardStr: String): Positions = {
    def charToPiece(c: Char) = {
      if (c == 'x') BlackPiece
      else if (c == 'o') WhitePiece
      else Empty
    }
    Vector(boardASCII.split("\n").drop(1).map(str => Vector(str: _*)
      .map(c => charToPiece(c))): _*)
  }
}
```

Here are test cases for `isLegalMove` method:

### Test for correct move's turn

```scala
class GoGameDefSpec extends WordSpec {

  trait NewGoGame5x5 extends GoGameDef {
    val rowCount = 5
    val colCount = 5
    val initialBoardRecord = Stream(Board())
  }

  "isLegalMove()" when {
    "at the start of the game" should {
      "be true if black's move" in {
        new NewGoGame5x5 {
          isLegalMove(Move(0, 0, BlackPiece), initialBoardRecord)
        }
      }

      "be false if white's move" in {
        new NewGoGame5x5 {
          assert(!isLegalMove(Move(0, 0, WhitePiece), initialBoardRecord))
        }
      }
    }

    ...
  }
}
```

### Test for moves with a coordinate outside of the board

```scala
"the move is outside the board's coordinate" must {
  "be false" in {
    new NewGoGame5x5 {
      assert(!isLegalMove(Move(-1, 2, BlackPiece), initialBoardRecord))
      assert(!isLegalMove(Move(3, -1, BlackPiece), initialBoardRecord))
      assert(!isLegalMove(Move(5, 1, BlackPiece), initialBoardRecord))
      assert(!isLegalMove(Move(4, 5, BlackPiece), initialBoardRecord))
    }
  }
}
```

### Test for moves on occupied positions

```scala
trait SelfCaptureGame1 extends StringParserGoGame {
  val boardASCII =
    """o
      |-x---
      |x----
      |-----
      |-----
      |-----
    """.stripMargin
}

"the move is on occupied position" must {
  "be false" in {
    new SelfCaptureGame1 {
      assert(!isLegalMove(Move(0, 1, WhitePiece), initialBoardRecord))
    }
  }
}
```

### Test for [Suicide rule](http://en.wikipedia.org/wiki/Go_%28game%29#Suicide)

```scala
"the move has no liberties (self-capturing)" must {
  "be false" in {
    new SelfCaptureGame1 {
      assert(!isLegalMove(Move(0, 0, WhitePiece), initialBoardRecord))
    }
  }
}

trait SelfCaptureGame2 extends StringParserGoGame {
  val boardASCII =
    """o
      |-xx--
      |x-ox-
      |xoox-
      |-xx--
      |-----
    """.stripMargin
}

"the move causes its connected group has no liberties (self-capturing)" must {
  "be false" in {
    new SelfCaptureGame2 {
      assert(!isLegalMove(Move(1, 1, WhitePiece), initialBoardRecord))
    }
  }
}

trait KoRuleGame extends StringParserGoGame {
  val boardASCII =
    """o
      |-----
      |-xo--
      |x-xo-
      |-xo--
      |-----
    """.stripMargin
}

"the move captures enemy's positions, even its position has no liberties" should {
  "be true" in {
    new KoRuleGame {
      assert(isLegalMove(Move(2, 1, WhitePiece), initialBoardRecord))
    }
  }
}
```

### Test for [ko rule](http://en.wikipedia.org/wiki/Go_%28game%29#The_ko_rule)

```scala
"the move causes the board state return to previous position" must {
  "be false" in {
    new KoRuleGame {
      val newBoardRecord = playMove(Move(2, 1, WhitePiece), initialBoardRecord)
      assert(!isLegalMove(Move(2, 2, BlackPiece), newBoardRecord))
    }
  }
}
```

## Implementation

For rule 1-3, I have a straightforward implementation:

```scala
def isLegalMove(move: Move, boardRecord: BoardRecord): Boolean = {
  require(boardRecord.nonEmpty)
  val currentBoard = boardRecord.head

  def isNextMove = move.piece == currentBoard.nextPiece
  def isOccupiedPos = currentBoard.positions(move.x)(move.y) != Empty
  def isSelfCapture: Boolean = ???
  def isSameAsPreviousState: Boolean = ???

  isNextMove && isInsideBoard(move.x, move.y) && !isOccupiedPos &&
      !isSelfCapture && !isSameAsPreviousState
}

protected def isInsideBoard(x: Int, y: Int): Boolean = {
  0 <= x && x < rowCount && 0 <= y && y < colCount
}
```

To check for suicide (or self-capture) rule, we need to determine whether a move causes its group to be captured. Thus I implement a function, called `getCapturedPieces`, which returns a set of captured positions of a type `Piece`, given a `Position` board:

```scala
/**
 + Given a board position, get all the captured pieces
 + which are same type of the input piece
 + @return A set contains positions of captured piece
 */
protected def getCapturedPieces(positions: Positions, piece: Piece): Set[(Int, Int)] = {
  require(piece != Empty)

  /**
   + Given a position (x, y), returns list of its neighbor positions
   */
  def neighbors(x: Int, y: Int): List[(Int, Int)] = {
    val neighborDisplacement = List((-1, 0), (1, 0), (0, -1), (0, 1))
    neighborDisplacement.map({ case (dx, dy)  => (x + dx, y + dy)})
      .filter({ case (i, j) => isInsideBoard(i, j) })
  }

  /**
   + Get all groups which are same type of the input piece
   */
  def getGroups: Set[Set[(Int, Int)]] = {

    /**
     + Find the group where piece at (x, y) belongs to
     */
    def findGroup(x: Int, y: Int) : Set[(Int, Int)] = {
      def visit(toVisit: Seq[(Int, Int)], visited: Set[(Int, Int)], group: Set[(Int, Int)]): Set[(Int, Int)] = {
        if (toVisit.isEmpty) group
        else {
          val pos = toVisit.head
          val toVisitFriendNeighbors = neighbors(pos._1, pos._2).toSeq.filter {
            case(i, j) => !visited.contains((i, j)) && positions(i)(j) == positions(pos._1)(pos._2)
          }
          visit(toVisit.tail ++ toVisitFriendNeighbors, visited + pos, group + pos)
        }
      }

      visit(Seq((x, y)), Set.empty, Set.empty)
    }

    def iterPos(x: Int, y: Int, visited: Set[(Int, Int)], groups: Set[Set[(Int, Int)]]): Set[Set[(Int, Int)]] = {
      def nextPos(x: Int, y: Int): (Int, Int) = {
        if (y < colCount - 1) (x, y + 1)
        else if (x < rowCount - 1) (x + 1, 0)
        else null
      }

      val group = {
        if (positions(x)(y) == piece && !visited.contains((x, y))) findGroup(x, y)
        else Set.empty[(Int, Int)]
      }

      nextPos(x, y) match {
        case (i, j) => iterPos(i, j, visited ++ group, groups + group)
        case _ => groups + group
      }
    }

    iterPos(0, 0, Set.empty, Set.empty)
  }

  /**
   + Whether a group is surrounded by enemy
   */
  def isSurrounded(group: Set[(Int, Int)]): Boolean = {
    def hasEmptyNeighbor(x: Int, y: Int): Boolean = {
      neighbors(x, y) exists {
        case (i, j) => positions(i)(j) == Empty }
    }
    !(group exists { case(x, y) => hasEmptyNeighbor(x, y) })
  }

  getGroups.filter(isSurrounded).flatten
}
```

The `getGroups` function uses a breadth-first search to find all connected-pieces or groups. The `isSurrounded` function tells us whether a group is surrouded by checking whether it has any `Empty` piece connected to it.

With the help of this function, we can implement the `isSelfCapture` function easily:

```scala
def isSelfCapture = {
  val canCaptureOpponent = getCapturedPieces(positionsWithMovePiece, move.piece.opponentPiece).nonEmpty
  if (canCaptureOpponent) false
  else {
    // if there is any captured piece of move.piece -> self-capture move
    val capturedOwnPieces = getCapturedPieces(positionsWithMovePiece, move.piece)
    capturedOwnPieces.nonEmpty
  }
}

lazy val positionsWithMovePiece = positionsWhenPlaceMove(move, currentBoard.positions)
```

First it checks whether the move can capture opponent's pieces. If yes, it means its groups will not be surrounded as there will be empty positions after taking out captured opponent's pieces. Otherwise, we check whether there is any captured group of the move's piece. If yes, it means the move is a sucide (or self-capture) move. The `positionsWhenPlaceMove` helper function returns a new board's position by simply placing the move's piece into its coordination.

To check for **ko** rule, I check the `boardRecord` for the previous state and check it with the board position after playing the move.

```scala
def isSameAsPreviousState: Boolean = boardRecord match {
  case (current#::previous#::xs) =>
    val newPosition = removeCapturedPieces(positionsWithMovePiece, move.piece.opponentPiece)
    previous.positions == newPosition
  case _ => false
}
```

The `removeCapturedPieces` helper function will take out captured pieces and return new board's position.

- - -

That is my implementation for a legal move checking in Go Game. For more details of the code, please visit its repository on my [GitHub](https://github.com/luugiathuy/gokata).

