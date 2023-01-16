---
title: "Bitboards in Rust"
date: 2022-09-28T21:46:16+05:30
draft: false
tags: ["bitboards", "chess", "chess-engine", "rust"]
---

---

### Introduction

Hello there! I haven't posted in a while because I was working on my chess engine. Specifically, representing the pieces as
bitboards. Now you may ask what a bitboard is. You can use bitboards to represent a chess board in a piece-centric manner. 
What is so special about bitboards then? Doesn't a piece list do the same thing? Well, representing a bitboard only requires a 
single unsigned 64 bit integer!

As taken from the [chessprogramming wiki](https://www.chessprogramming.org/):

> To represent the board we typically need one bitboard for each piece-type and color -
	likely encapsulated inside a class or structure, or as an array of bitboards as part of a position object.
	A one-bit inside a bitboard implies the existence of a piece of this piece-type on a certain square - one to
	one associated by the bit-position.


For example, a bitboard containing all of white's pawns in the starting position would look like this:

```diff
. . . . . . . .
. . . . . . . .
. . . . . . . .
. . . . . . . .
. . . . . . . .
. . . . . . . .
1 1 1 1 1 1 1 1
. . . . . . . .
```

As you can observe, the squares containing the pawns have a value of 1 and others have a value of 0. A bitboard representing a
particular piece will have 1 in the squares the piece occupies and 0 in all the other squares.

Before proceding further, I'd like to shed light on the fact that I'm fairly new to the chess engine development scene, so all suggestions
are welcome!

### Implementation

I decided to go with representing all the bitboards in a single position as a `struct`.

```rust
#[derive(Debug, Clone)]
struct BitPos {
	wp: BitBoard, // white pawns
	wn: BitBoard, // white knights
	wb: BitBoard, // white bishops
	wr: BitBoard, // white rooks
	wq: BitBoard, // white queen
	wk: BitBoard, // white king
	bp: BitBoard, // black pawns
	bn: BitBoard, // black knights
	bb: BitBoard, // black bishops
	br: BitBoard, // black rooks
	bq: BitBoard, // black queen
	bk: BitBoard, // black king
}
```

BitBoard is a struct containing a u64.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct BitBoard(pub u64);
```

Then I wrote a simple function to generate a bitboard from my piece list array 
(see [fen-string-parsing-in-rust](https://rv178.is-a.dev/posts/fen-string-parsing-in-rust/))

```rust
fn gen(pieces: &mut [Option<Piece>; 64], compare: char) -> Self {
    let mut bin_str = String::new();
    // reverse pieces array
    pieces.reverse();

    for piece in pieces {
        if let Some(piece) = piece {
            if !(piece.symbol == compare) {
                bin_str.push('0');
            } else {
                bin_str.push('1');
            }
        } else {
            bin_str.push('0');
        }
    }

    // convert binary string to decimal (u64)
    Self(u64::from_str_radix(&bin_str, 2).unwrap())
}
```

And another function to print out the bitboard:

```rust
fn print(&self) {
    println!("Value: {}", &self.0);
    println!();
    let mut x = 8;
    for rank in 0..8 {
        print!("\x1b[34m{}\x1b[0m  ", x);
        x -= 1;
        for file in 0..8 {
            let square = rank * 8 + file;
            if self.0 & (1 << square) >= 1 {
                print!("\x1b[1m1\x1b[0m ");
            } else {
                print!("\x1b[38;5;8m0\x1b[0m ");
            }
        }
        println!();
    }
    println!();
    println!("\x1b[34m   a b c d e f g h\x1b[0m");
    println!();
}
```

And some other functions for convenience:

```rust
// generate bitboard from square
fn from_sq(square: Square) -> Self {
    Self(1 << square as u8)
}
// init empty bitboard
fn empty() -> Self {
    Self(0)
}
// set bit at given square (0 -> 1)
fn set_bit(&mut self, square: Square) {
    self.0 |= BitBoard::from_sq(square).0;
}
// toggle bit at given square
fn toggle_bit(&mut self, square: Square) {
    self.0 ^= BitBoard::from_sq(square).0;
}
```

What fascinated me is the fact that how easy it was to perform actions on bitboards.

I also made an enum to represent the squares (just for convenience and better code readability).

```rust
// to represent squares
#[rustfmt::skip]
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
enum Square {
    A8, B8, C8, D8, E8, F8, G8, H8,
    A7, B7, C7, D7, E7, F7, G7, H7,
    A6, B6, C6, D6, E6, F6, G6, H6,
    A5, B5, C5, D5, E5, F5, G5, H5,
    A4, B4, C4, D4, E4, F4, G4, H4,
    A3, B3, C3, D3, E3, F3, G3, H3,
    A2, B2, C2, D2, E2, F2, G2, H2,
    A1, B1, C1, D1, E1, F1, G1, H1,
}
```

### Piece attack generation

I found this very interesting. I still haven't finished working on move generation for sliding pieces at the time of writing this.

Anyways, for generating pawn attacks, you can shift the bitboard by 7 and 9 in specific directions to achieve the desired result.
That is, capturing diagonally.

For example:
```rust
const NOT_A_FILE: BitBoard = BitBoard(18374403900871474942);
const NOT_H_FILE: BitBoard = BitBoard(9187201950435737471);

pub fn east(board: BitBoard, colour: Colour) -> BitBoard {
    match colour {
        Colour::White => BitBoard((board.0 >> 9) & NOT_H_FILE.0),
        Colour::Black => BitBoard((board.0 << 9) & NOT_A_FILE.0),
        Colour::Undefined => exit(1),
    }
}

pub fn west(board: BitBoard, colour: Colour) -> BitBoard {
    match colour {
        Colour::White => BitBoard((board.0 >> 7) & NOT_A_FILE.0),
        Colour::Black => BitBoard((board.0 << 7) & NOT_H_FILE.0),
        Colour::Undefined => exit(1),
    }
}
```

I also a made a function for convenience (lookup pawn attacks for a particular square):

```rust
pub fn lookup(square: Square, colour: Colour) -> BitBoard {
    let board = BitBoard::from_sq(square);

    match colour {
        Colour::White => BitBoard(east(board, Colour::White).0 | west(board, Colour::White).0),
        Colour::Black => BitBoard(east(board, Colour::Black).0 | west(board, Colour::Black).0),
        Colour::Undefined => exit(1),
    }
}
```

And another one for all pawns:

```rust
pub fn all(board: BitBoard, colour: Colour) -> BitBoard {
    match colour {
        Colour::White => BitBoard(east(board, Colour::White).0 | west(board, Colour::White).0),
        Colour::Black => BitBoard(east(board, Colour::Black).0 | west(board, Colour::Black).0),
        Colour::Undefined => exit(1),
    }
}
```

Attacks can be generated in the same manner for knights and kings:

#### Knights

```diff
        noNoWe    noNoEa
            +15  +17
             |     |
noWeWe  +6 __|     |__+10  noEaEa
              \   /
               >0<
           __ /   \ __
soWeWe -10   |     |   -6  soEaEa
             |     |
            -17  -15
        soSoWe    soSoEa
```

```rust
const NOT_HG_FILE: BitBoard = BitBoard(4557430888798830399);
const NOT_AB_FILE: BitBoard = BitBoard(18229723555195321596);

pub fn no_no_east(board: BitBoard) -> BitBoard {
    BitBoard((board.0 << 17) & NOT_A_FILE.0)
}

pub fn no_no_west(board: BitBoard) -> BitBoard {
    BitBoard((board.0 << 15) & NOT_H_FILE.0)
}

pub fn so_so_east(board: BitBoard) -> BitBoard {
    BitBoard((board.0 >> 15) & NOT_A_FILE.0)
}

pub fn so_so_west(board: BitBoard) -> BitBoard {
    BitBoard((board.0 >> 17) & NOT_H_FILE.0)
}

pub fn no_ea_east(board: BitBoard) -> BitBoard {
    BitBoard((board.0 << 10) & NOT_AB_FILE.0)
}

pub fn so_ea_east(board: BitBoard) -> BitBoard {
    BitBoard((board.0 >> 6) & NOT_AB_FILE.0)
}

pub fn no_we_west(board: BitBoard) -> BitBoard {
    BitBoard((board.0 << 6) & NOT_HG_FILE.0)
}

pub fn so_we_west(board: BitBoard) -> BitBoard {
    BitBoard((board.0 >> 10) & NOT_HG_FILE.0)
}

// and then just get the union of all the other boards
pub fn knight(sq: Square) -> BitBoard {
    let board = BitBoard::from_sq(sq);

    let no_no_east = knight::no_no_east(board);
    let no_no_west = knight::no_no_west(board);
    let so_so_east = knight::so_so_east(board);
    let so_so_west = knight::so_so_west(board);
    let no_ea_east = knight::no_ea_east(board);
    let so_ea_east = knight::so_ea_east(board);
    let no_we_west = knight::no_we_west(board);
    let so_we_west = knight::so_we_west(board);

    BitBoard(
        no_no_east.0
            | no_no_west.0
            | so_so_east.0
            | so_so_west.0
            | no_ea_east.0
            | so_ea_east.0
            | no_we_west.0
            | so_we_west.0,
    )
}
```

Result (knight on e4):

```md
8  . . . . . . . . 
7  . . . . . . . . 
6  . . . 1 . 1 . . 
5  . . 1 . . . 1 . 
4  . . . . . . . . 
3  . . 1 . . . 1 . 
2  . . . 1 . 1 . . 
1  . . . . . . . . 

   a b c d e f g h
```

#### King

```rust

// north
pub fn no(board: BitBoard) -> BitBoard {
    BitBoard(board.0 << 8)
}

// south
pub fn so(board: BitBoard) -> BitBoard {
    BitBoard(board.0 >> 8)
}

// east
pub fn ea(board: BitBoard) -> BitBoard {
    BitBoard((board.0 << 1) & NOT_A_FILE.0)
}

// west
pub fn we(board: BitBoard) -> BitBoard {
    BitBoard((board.0 >> 1) & NOT_H_FILE.0)
}

// north east
pub fn no_ea(board: BitBoard) -> BitBoard {
    BitBoard((board.0 << 9) & NOT_A_FILE.0)
}

// north west
pub fn no_we(board: BitBoard) -> BitBoard {
    BitBoard((board.0 << 7) & NOT_H_FILE.0)
}

// south east
pub fn so_ea(board: BitBoard) -> BitBoard {
    BitBoard((board.0 >> 7) & NOT_A_FILE.0)
}

// south west
pub fn so_we(board: BitBoard) -> BitBoard {
    BitBoard((board.0 >> 9) & NOT_H_FILE.0)
}

pub fn king(sq: Square) -> BitBoard {
    let board = BitBoard::from_sq(sq);

    let no = king::no(board);
    let so = king::so(board);
    let ea = king::ea(board);
    let we = king::we(board);
    let no_ea = king::no_ea(board);
    let no_we = king::no_we(board);
    let so_ea = king::so_ea(board);
    let so_we = king::so_we(board);

    BitBoard(no.0 | so.0 | ea.0 | we.0 | no_ea.0 | no_we.0 | so_ea.0 | so_we.0)
}
```

Result (king on e4):

```md
8  . . . . . . . . 
7  . . . . . . . . 
6  . . . . . . . . 
5  . . . 1 1 1 . . 
4  . . . 1 . 1 . . 
3  . . . 1 1 1 . . 
2  . . . . . . . . 
1  . . . . . . . . 

   a b c d e f g h
```

That's it for this blog, see you soon!

### References

- [chessprogramming.org/Bitboards](https://www.chessprogramming.org/Bitboards)
- [Chess engine in C](https://www.youtube.com/playlist?list=PLmN0neTso3Jxh8ZIylk74JpwfiWNI76Cs)
