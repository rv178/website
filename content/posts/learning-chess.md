---
title: "Learning Chess"
date: 2022-06-03T12:10:51+05:30
draft: false
tags: ["chess", "chess engines", "lichess"]
---

---

So a while ago after being absolutely destroyed by a friend in a game of chess, I decided to actually learn the game.

### What's chess?

Chess is a popular game of strategy where two players play against each other.
It involves a chess board containing 64 squares and 16 pieces on each side.

The players have a pair of rooks, bishops and knights. They also have eight pawns, a queen and a king. The game is over when you deliver a check
(threatening to capture the king of the opposite side) which the enemy king cannot escape.

### Getting better

I started by playing games (yeah too obvious) and registering on [lichess.org](https://lichess.org), an amazing open source chess
website. Most of these games were against my friends because after playing a rated game I lost horribly to an opponent (got checkmated in like 9 moves).

After playing a few games I decided to check out [lichess.org/learn](https://lichess.org/learn#/) which proved to be a great learning experience.

### Chess engines

I didn't become a grandmaster after finishing it (unfortunately) so I played more games, particularly against the [Stockfish chess engine](https://stockfishchess.org/)
because my friends wouldn't agree to play with me.

Around this time I was extremely interested in making a chess engine (inspired by stockfish) and I created a project in Rust, but eventually gave up because it was a lot of work.
I had managed to create a board implementation but after trying to implement FEN string parsing I gave up midway.

However for the fellow devs who are reading this, [chessprogramming.org](https://www.chessprogramming.org/Main_Page) covers a lot of information regarding engine development in detail.

### Status

Currently I can beat Stockfish Level 4 (1700) on lichess.org and I managed to draw a match with Level 5 (2000).

I still suck at chess but I hope to become better in the future. It's a very beautiful game and I enjoy playing every match.
