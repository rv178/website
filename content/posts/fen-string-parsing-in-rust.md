---
title: "FEN String Parsing in Rust"
date: 2022-06-29T14:54:23+05:30
draft: false
tags: ["fen", "chess", "rust"]
---

---

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
    pub en_passant: Option<Vec<String>>,
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

Since the input can either be `-` or multiple squares like `e4e5g6` etc., I wanted a vector that contained each square name.

So basically,

> `e4e5g6` => `["e4", "e5", "g6"]`

```rust
fn parse_en_passant_squares(input: &str) -> Vec<String> {
    let re = Regex::new(r"^-|[a-g]\d$").unwrap(); // check if input is valid
    if re.is_match(input) {
        let mut char_vec: Vec<String> = Vec::new(); // for storing all characters
        let mut split_vec: Vec<String> = Vec::new(); // for joining the characters
        let mut en_passant_vec: Vec<String> = Vec::new(); // our final vec

        let split_string = input.split(|c: char| c.is_whitespace());
        for s in split_string {
            for c in s.chars() {
                char_vec.push(c.to_string());
            }
        }
        for i in 0..char_vec.len() {
            if i % 2 == 0 { // check if index is divisible by 2 and then push two times
                split_vec.push(char_vec[i].to_string());
                split_vec.push(char_vec[i + 1].to_string());
                en_passant_vec.push(split_vec.join("")); // join split_vec and push it to our en passant vec
                split_vec.clear(); // clear split_vec to repeat the process
            }
        }

        en_passant_vec
    } else {
        println!("Invalid FEN string: Failed to parse en passant squares.");
        exit(1);
    }
}
```

```rust
fn en_passant(input: &str) -> Option<Vec<String>> {
    if input == "-" {
        None
    } else if input.chars().all(char::is_whitespace) {
        None
    } else {
        Some(parse_en_passant_squares(input))
    }
}
```

I'm just an amateur rust developer so there might be 100000 different ways of doing this which are more efficient. For now, it works 😂.

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

I also decided to add this function to print out the game state. Since it's a 1d array I wanted to convert it to a 2d array for printing it.

So basically what I did was:

```rust
let mut symbol_vec: Vec<char> = Vec::new(); // 1d vec that contains piece chars

for i in 0..game_state.pieces.len() {
    if game_state.pieces[i] == None {
        symbol_vec.push(' ');
    } else {
        symbol_vec.push(game_state.pieces[i].as_ref().unwrap().symbol);
    }
}
```

Then I used a function to make a new 2d vector that contains the piece chars.

```rust
fn vec_to_2d_vec(vec: Vec<char>) -> Vec<Vec<char>> {
    let mut vec_2d: Vec<Vec<char>> = Vec::new();
    let mut vec_1d: Vec<char> = Vec::new();
    for i in 0..vec.len() {
        vec_1d.push(vec[i]);
		if vec_1d.len() == 8 {
			vec_2d.push(vec_1d);
			vec_1d = Vec::new();
        }
    }
    vec_2d
}
```

And finally, print it all out.

```rust
let board = vec_to_2d_vec(symbol_vec);

fn print_board(array: &Vec<Vec<char>>) -> String {
    let mut x = 9;
    println!("+---+---+---+---+---+---+---+---+");
    let mut buf = String::new();
    for (_y, row) in array.iter().enumerate() {
        for (_x, col) in row.iter().enumerate() {
            let p_info = format!("| {} ", col);
            buf.push_str(&p_info);
        }
        x = x - 1;

        let ranks = format!("| {} \n", x);
        buf.push_str(&ranks);

        buf.push_str("+---+---+---+---+---+---+---+---+\n");
    }
    buf
}

print!("{}", print_board(&board));
println!("  a   b   c   d   e   f   g   h  \n");
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
