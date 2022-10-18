---
title: "Hyperbola Quintessence in Rust"
date: 2022-10-18T21:53:57+05:30
draft: false
tags: ["rust", "chess", "chess engine", "bitboards"]
---

---

Welcome to part 3 - or 4, (I only know how to count up to 3) of my dev-blog series on writing a chess engine. Like always, suggestions are welcome!

### Hyperbola quintessence?

This is the definition from [chessprogramming.org](https://www.chessprogramming.org/Hyperbola_Quintessence):

> Hyperbola Quintessence applies the o^(o-2r)-trick also for vertical or diagonal negative Rays - by reversing the bit-order
 of up to one bit per rank or byte with a vertical flip aka x86-64 bswap.

Why did I go for this approach? Because I thought [magic bitboards](https://www.chessprogramming.org/Magic_Bitboards) were too
hard for me.

### Implementation

```rust
// hyperbola quintessence
pub fn hyp_quint(sq: Square, occ: BitBoard, mask: u64) -> BitBoard {
    let mut forward = occ.. & mask;
    let mut reverse = forward.reverse_bits();

    forward = forward.wrapping_sub(BitBoard::from_sq(sq).0);
    reverse = reverse.wrapping_sub(BitBoard::from_sq(sq).0.reverse_bits());
    forward ^= reverse.reverse_bits();
    forward &= mask;

    BitBoard(forward)
}
```
Where `sq` is the square the piece is in, `occ` is the occupancy bitboard, and `mask` is the attack mask. Why use `wrapping_sub()`
instead of the `-=` operator? I found that the code sometimes panics if the variable is out of bounds.

As I already covered in my previous blog:

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

Square is an enum. The occupancy bitboard gives us the occupied squares (where the rays are blocked). As for the mask, here's an
example:

Let's say we wish to get the attacks in one file. For this, let's assume a rook is on E4:

```
Hex: 1000000000
Value: 68719476736

8  . . . . . . . . 
7  . . . . . . . . 
6  . . . . . . . . 
5  . . . . . . . . 
4  . . . . 1 . . . 
3  . . . . . . . . 
2  . . . . . . . . 
1  . . . . . . . . 

   a b c d e f g h
```

And C4 and G4 are occupied:

```

Hex: 4400000000
Value: 292057776128

8  . . . . . . . . 
7  . . . . . . . . 
6  . . . . . . . . 
5  . . . . . . . . 
4  . . 1 . . . 1 . 
3  . . . . . . . . 
2  . . . . . . . . 
1  . . . . . . . . 

   a b c d e f g h
```

Now the mask for generating attack on this specific rank would be a bitboard of the 4th rank. That is,

```

Hex: ff00000000
Value: 1095216660480

8  . . . . . . . . 
7  . . . . . . . . 
6  . . . . . . . . 
5  . . . . . . . . 
4  1 1 1 1 1 1 1 1 
3  . . . . . . . . 
2  . . . . . . . . 
1  . . . . . . . . 

   a b c d e f g h
```

Now if we execute the hyperbola quintessence function on this, we get all possible attacks for the rook on the 4th rank.

```rust
hyp_quint(Square::E4, r, RANKS[4].0).print();
```

```
Hex: 6c00000000
Value: 463856467968

8  . . . . . . . . 
7  . . . . . . . . 
6  . . . . . . . . 
5  . . . . . . . . 
4  . . 1 1 . 1 1 . 
3  . . . . . . . . 
2  . . . . . . . . 
1  . . . . . . . . 

   a b c d e f g h
```

This is the intended bitboard denoting all possible rank-attacks of a rook on E4 with respect to the occupancy bitboard. Note
that we also generate an attack on the blocked bits, as they can be captures (unless they're friendly pieces).

We can do the same for generating rook attacks both vertically and horizontally.

```rust
pub fn rook(sq: Square, occ: BitBoard) -> BitBoard {
    let tr = sq as usize / 8;
    let tf = sq as usize % 8;

    BitBoard(hyp_quint(sq, occ, FILES[tf].0).0 | hyp_quint(sq, occ, RANKS[tr].0).0)
}
```

Where `tr` is the target rank, and `tf` is the target file (for getting the value of file/rank mask).

```rust
pub const RANKS: [BitBoard; 8] = [
    BitBoard(255),                  // 8th rank
    BitBoard(65280),                // 7th rank
    BitBoard(16711680),             // 6th rank
    BitBoard(4278190080),           // 5th rank
    BitBoard(1095216660480),        // 4th rank
    BitBoard(280375465082880),      // 3rd rank
    BitBoard(71776119061217280),    // 2nd rank
    BitBoard(18374686479671623680), // 1st rank
];

pub const FILES: [BitBoard; 8] = [
    BitBoard(72340172838076673),   // a file
    BitBoard(144680345676153346),  // b file
    BitBoard(289360691352306692),  // c file
    BitBoard(578721382704613384),  // d file
    BitBoard(1157442765409226768), // e file
    BitBoard(2314885530818453536), // f file
    BitBoard(4629771061636907072), // g file
    BitBoard(9259542123273814144), // h file
];
```

For bishops, we can do the same but with diagonal and anti-diagonal masks.

```rust
// diagonal masks
pub const DIAG: [BitBoard; 15] = [
    BitBoard(0x80),
    BitBoard(0x8040),
    BitBoard(0x804020),
    BitBoard(0x80402010),
    BitBoard(0x8040201008),
    BitBoard(0x804020100804),
    BitBoard(0x80402010080402),
    BitBoard(0x8040201008040201),
    BitBoard(0x4020100804020100),
    BitBoard(0x2010080402010000),
    BitBoard(0x1008040201000000),
    BitBoard(0x804020100000000),
    BitBoard(0x402010000000000),
    BitBoard(0x201000000000000),
    BitBoard(0x100000000000000),
];

// anti-diagonal masks
pub const ANTI_DIAG: [BitBoard; 15] = [
    BitBoard(0x1),
    BitBoard(0x102),
    BitBoard(0x10204),
    BitBoard(0x1020408),
    BitBoard(0x102040810),
    BitBoard(0x10204081020),
    BitBoard(0x1020408102040),
    BitBoard(0x102040810204080),
    BitBoard(0x204081020408000),
    BitBoard(0x408102040800000),
    BitBoard(0x810204080000000),
    BitBoard(0x1020408000000000),
    BitBoard(0x2040800000000000),
    BitBoard(0x4080000000000000),
    BitBoard(0x8000000000000000),
];
```

```rust
pub fn bishop(sq: Square, occ: BitBoard) -> BitBoard {
    let tr = sq as usize / 8;
    let tf = sq as usize % 8;

    let diag_index: usize = 7 + tr - tf;
    let anti_diag_index: usize = tr + tf;

    BitBoard(
        hyp_quint(sq, occ, DIAG[diag_index].0).0
            | hyp_quint(sq, occ, ANTI_DIAG[anti_diag_index].0).0,
    )
}
```

And for generating queen attacks, it's even easier since we can take the union of the bishop and rook attack bitboards:

```rust
pub fn queen(sq: Square, occ: BitBoard) -> BitBoard {
    BitBoard(rook(sq, occ).0 | bishop(sq, occ).0)
}
```

That's it for this blog, see you soon!
