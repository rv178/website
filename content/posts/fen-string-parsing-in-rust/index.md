---
title: "FEN String Parsing in Rust"
date: 2022-06-29T14:54:23+05:30
draft: false
tags: ["Rust", "Chess Engine"]

cover:
    image: "cover.png"
    alt: "Description of image"
    caption: "FEN string to chess position"
    relative: true
---

---

## Introduction

I **finally** finished the FEN string parsing part of my chess engine after
days of procrastinating and I just wanted to share my experience with everyone.

So a FEN string looks like this:

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

This is the FEN string of the starting position in standard chess. It looks like random crap at first glance but
this string conveys a lot of information.

So let's divide it into parts.

### Pieces

The first part conveys information regarding piece placement. Each letter represents a piece on each rank of the board, and numbers
denote empty spaces.

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
```

Let's look at an example from chess.com here:

![chess.com-example](https://images.chesscomfiles.com/uploads/v1/images_users/tiny_mce/pdrpnht/phpffYq5N.png)

As you can observe lower case characters are for black pieces and upper case characters are for white pieces.

Now, moving on to the second part.

### Side to move

The second part consists of just a letter which can either be `b` or `w`.

Here we can observe that it's `w`.

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR | w | KQkq | - | 0 | 1
```

That means it's white's turn to move, and `b` means it's black's turn to move.

### Castling ability

The third part contains information regarding castling ability of both sides.

`K` - White can castle **kingside**.

`k` - Black can castle **kingside**.

`Q` - White can castle **queenside**.

`q` - Black can castle **queenside**.

If no sides can castle, `-` is used like so:

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w - - 0 1
```

### En passant squares

The next part gives us information if any en passant squares are possible. If there are no en passant squares, `-` is used as demonstrated in this example. If there are any, then the co-ordinates of that
square is used.

### Half move clock and full move counter

The next two parts give us information regarding the number of halfmoves and fullmoves completed. Here you can observe the half move count is `0`
and the full move count is `1`.

## Parsing the string

To track the current game status, I made a struct called `GameStatus`.

```rust
pub struct GameStatus {
    pub pieces: Vec<Option<Piece>>,
    pub side_to_move: Colour,
    pub castling_id: [bool; 4],
    pub en_passant: Option<Vec<Square>>,
    pub half_move_clock: u32,
    pub full_move_count: u32,
}
```

What I wanted to do here was make functions for each variable and then assign it to the fields in the struct.

For parsing the side to move I first created an enumerator called `Colour`.

```rust
pub enum Colour {
    White,
    Black,
    Undefined,
}
```

White and Black are both sides, and undefined is used for error handling if the input is invalid.

I then created a function that will return a value based on the input given. I really love rust's match syntax 😄.

```rust
fn active_side(input: &str) -> Colour {
    match input {
        "w" => Colour::White,
        "b" => Colour::Black,
        _ => Colour::Undefined,
    }
}

```

```rust
if colour == Colour::Undefined {
    println!("Invalid FEN string: Failed to parse active colour.");
    exit(1);
}
```

For parsing castling ability I decided to make an array of bools that would contain values corresponding to the info parsed from
the string.

> `[true, true, true, true]` if input is `KQkq` (all sides can castle both ways).

Index `0` checks if the white king can castle kingside. ie, `K` and so on.

Index `1` => can white king castle queenside?

Index `2` => can black king castle kingside?

Index `3` => can black king castle queenside?

So I made a function that returns this array of bools.

```rust
fn castling_ability(input: &str) -> [bool; 4] {
    if input == "-" {
        [false, false, false, false]
    } else {
        let mut castling_id = [false; 4];
        let mut castling_id_str = input.to_string();
        castling_id_str.retain(|c| c != '-');
        for c in castling_id_str.chars() {
            match c {
                'K' => castling_id[0] = true,
                'Q' => castling_id[1] = true,
                'k' => castling_id[2] = true,
                'q' => castling_id[3] = true,
                _ => {
                    println!("Invalid FEN string: Failed to parse castling ability.");
                    exit(1);
                }
            }
        }
        castling_id
    }
}
```

For parsing en passant squares the solution I came up with was rather stupid.

Since the input can either be `-` or multiple squares like `e4e5g6` etc., I wanted a vector that contained each square.

So basically,

> `e4e5g6` => `[Square::E4, Square::E5, Square::G6]`

(Square is an enum, this change was made later. Check [this blog](https://rv178.is-a.dev/posts/bitboards-in-rust/) for more info. Yes, I updated this blog :D)

```rust
fn en_passant(input: &str) -> Option<Vec<Square>> {
    if input.chars().all(char::is_whitespace) | input.contains('-') {
        None
    } else {
        let chars = input.chars().collect::<Vec<char>>();

        if chars.len() % 2 == 0 {
            let mut ep_vec = Vec::new();
            let mut sq;

            for i in 0..chars.len() {
                if i % 2 == 0 {
                    let valid_chars = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h'];
                    let mut invalid = true;

                    for c in valid_chars.iter() {
                        if chars[i] == *c {
                            invalid = false;
                        }
                    }

                    if i < chars.len() - 1 {
                        if !chars[i].is_alphabetic() | !chars[i + 1].is_numeric() {
                            fen_log!("Invalid FEN string: Failed to parse en passant square.");
                            exit(1);
                        }

                        if chars[i + 1]
                            .to_digit(10)
                            .expect("Could not parse character to digit (en passant)")
                            > 8
                        {
                            invalid = true;
                        }
                    }

                    if invalid {
                        fen_log!("Invalid FEN string: Failed to parse en passant square.");
                        exit(1);
                    }

                    let file;

                    match chars[i] {
                        'a' => file = 0,
                        'b' => file = 1,
                        'c' => file = 2,
                        'd' => file = 3,
                        'e' => file = 4,
                        'f' => file = 5,
                        'g' => file = 6,
                        'h' => file = 7,
                        _ => {
                            fen_log!("Invalid FEN string: Failed to parse en passant square.");
                            exit(1);
                        }
                    }

                    let rank;

                    match chars[i + 1] {
                        '1' => rank = 7,
                        '2' => rank = 6,
                        '3' => rank = 5,
                        '4' => rank = 4,
                        '5' => rank = 3,
                        '6' => rank = 2,
                        '7' => rank = 1,
                        '8' => rank = 0,
                        _ => {
                            fen_log!("Invalid FEN string: Failed to parse en passant square.");
                            exit(1);
                        }
                    }

                    sq = rank * 8 + file;

                    ep_vec.push(match_u32_to_sq(sq as u32));
                }
            }

            Some(ep_vec)
        } else {
            fen_log!("Invalid FEN string: Failed to parse en passant square.");
            exit(1);
        }
    }
}
```

Parsing the halfmove and fullmove counts were rather easy, and I just had to return a u32 from the string input.

```rust
fn halfmove_clock(input: &str) -> u32 {
    let mut halfmove_clock = String::new();
    for c in input.chars() {
        if c.is_digit(10) {
            halfmove_clock.push(c);
        }
    }
    halfmove_clock.parse::<u32>().unwrap()
}

fn fullmove_count(input: &str) -> u32 {
    let mut fullmove_clock = String::new();
    for c in input.chars() {
        if c.is_digit(10) {
            fullmove_clock.push(c);
        }
    }
    fullmove_clock.parse::<u32>().unwrap()
}
```

However for piece placement parsing, I saw a really good implementation of it in the [fen](https://lib.rs/crates/fen) crate so I decided to just
joink it 😔.

There's a struct called `Piece` that stores the type, colour and symbol (added by me to print out pieces in the board).

```rust
pub struct Piece {
    pub kind: Kind,
    pub colour: Colour,
    pub symbol: char,
}
```

And an enum `Kind`:

```rust
pub enum Kind {
    King,
    Queen,
    Bishop,
    Knight,
    Rook,
    Pawn,
}
```

And according to the input given, a value is returned that is later pushed to a vector.

```rust
let (color, kind, symbol) = match piece_char {
    'P' => (Colour::White, Kind::Pawn, 'P'),
    'N' => (Colour::White, Kind::Knight, 'N'),
    'B' => (Colour::White, Kind::Bishop, 'B'),
    'R' => (Colour::White, Kind::Rook, 'R'),
    'Q' => (Colour::White, Kind::Queen, 'Q'),
    'K' => (Colour::White, Kind::King, 'K'),
    'p' => (Colour::Black, Kind::Pawn, 'p'),
    'n' => (Colour::Black, Kind::Knight, 'n'),
	'b' => (Colour::Black, Kind::Bishop, 'b'),
    'r' => (Colour::Black, Kind::Rook, 'r'),
    'q' => (Colour::Black, Kind::Queen, 'q'),
	'k' => (Colour::Black, Kind::King, 'k'),
    _ => return None,
};
```

You can check out the crate's code for more info.

I want to write my own implementation too but I'm too lazy and it just works.

I also decided to add this function to print out the game state. So basically what I did was:

```rust
pub fn print_board(game_state: &GameStatus) {
    let mut board: Vec<char> = Vec::new();

    for piece in game_state.pieces {
        if piece == None {
            board.push(' ');
        } else {
            board.push(piece.as_ref().unwrap().symbol);
        }
    }

    let mut x = 8;
    println!("+---+---+---+---+---+---+---+---+");
    for rank in 0..8 {
        x -= 1;
        for file in 0..8 {
            let square = rank * 8 + file;
            if board[square] == ' ' {
                print!("|   ");
            } else {
                print!("| {} ", board[square]);
            }
        }
        print!("| {} \n", x + 1);
        println!("+---+---+---+---+---+---+---+---+");
    }

    println!("  a   b   c   d   e   f   g   h  \n");
}
```

Input: `rnbqkbnr/pp1ppppp/8/2p5/4P3/5N2/PPPP1PPP/PNBQKB1R b KQkq - 1 2`

Output:

```md
+---+---+---+---+---+---+---+---+
| r | n | b | q | k | b | n | r | 8
+---+---+---+---+---+---+---+---+
| p | p |   | p | p | p | p | p | 7
+---+---+---+---+---+---+---+---+
|   |   |   |   |   |   |   |   | 6
+---+---+---+---+---+---+---+---+
|   |   | p |   |   |   |   |   | 5
+---+---+---+---+---+---+---+---+
|   |   |   |   | P |   |   |   | 4
+---+---+---+---+---+---+---+---+
|   |   |   |   |   | N |   |   | 3
+---+---+---+---+---+---+---+---+
| P | P | P | P |   | P | P | P | 2
+---+---+---+---+---+---+---+---+
| P | N | B | Q | K | B |   | R | 1
+---+---+---+---+---+---+---+---+
  a   b   c   d   e   f   g   h
```

Anyways that's it for this blog see you soon 👋.
