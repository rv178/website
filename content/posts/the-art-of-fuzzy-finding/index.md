---
title: "The Art of Fuzzy Finding"
date: 2026-06-22T12:29:33+05:30
draft: true
tags: ["Go"]
summary: "Writing a SIMD accelerated fuzzy finder that uses the Smith-Waterman algorithm."

cover:
    image: "cover.png"
    alt: "Cover"
    caption: "Writing a SIMD accelerated fuzzy finder that uses the Smith-Waterman algorithm."
    relative: true
---

---

## Introduction

The other day, my friend and I were having a particularly unusual conversation which pertained 
to the abstract idea of "employment" (foreign to many, including myself).

Anyhow, we were talking about the things we were working on, and he mentioned that he was building a bioinformatics sequence alignment
toolkit (mainly to understand SIMD).

The crux of the conversation was that sequence aligners in bioinformatics borrow the concept of [edit distance](https://en.wikipedia.org/wiki/Edit_distance)
which is a way of quantifying how dissimilar two strings are (in their case, taking sequences of DNA, RNA, etc. and trying to align and analyze the patterns).

This is an example taken from [Wikipedia](https://en.wikipedia.org/wiki/Levenshtein_distance):

> For example, the **Levenshtein distance** between "kitten" and "sitting" is **3**, since the following 3 edits change one into the other, 
> and there is no way to do it with fewer than 3 edits:
> 
>       1. kitten → sitten (substitution of "s" for "k"),
>       2. sitten → sittin (substitution of "i" for "e"), 
>       3. sittin → sitting (insertion of "g" at the end).
> A simple example of a deletion can be seen with "uninformed" and "uniformed" which have a distance of 1:
>
>       1. uninformed → uniformed (deletion of "n").

The next morning, I was wondering if this could be used for fuzzy finding. However, this particular algorithm is more suited for programs like
spell checkers (it is a [global alignment](https://en.wikipedia.org/wiki/Sequence_alignment#Global_and_local_alignments) algorithm).

The funny part is that fuzzy searching (in tools like [fff](https://github.com/dmtrKovalenko/fff)) is also done by the same algorithm used for local sequence alignment in bioinformatics.
After checking out [frizbee](https://github.com/saghen/frizbee/#smith-waterman), I decided to write my own fuzzy searching tool implementing
the Smith-Waterman algorithm.

...but why?

Fuzzy finding is a tool we often take for granted (at least in my workflow), whether it be sifting through directories, or opening files,
or even just searching through things in general. I wanted to know the concept behind it.


## Longest Common Subsequence

Let's consider a simpler problem space.

Consider two strings, `s1 = "ccdegf"` and `s2 = "def"`.

We need to find the longest sequence of characters that appears in both strings in the same relative order.

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c}
  \textcolor{#81a1c1}{\texttt{\text{ccdegf}}} \hspace{0.5em}
  & \; \texttt{\text{c}} & \; 
  \texttt{\text{c}} & \; 
  \textcolor{#bf616a}{\texttt{\text{d}}} & \; 
  \textcolor{#bf616a}{\texttt{\text{e}}} & \; 
  \texttt{\text{g}} & \; 
  \textcolor{#bf616a}{\texttt{\text{f}}} & \; 
\end{array}
$$

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c}
  \textcolor{#81a1c1}{\texttt{\text{def}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{d}}} & \; 
  \textcolor{#bf616a}{\texttt{\text{e}}} & \; 
  \textcolor{#bf616a}{\texttt{\text{f}}} & \; 
\end{array}
$$

The LCS here is `def`.

In the first string, *not all* the characters are adjacent to each other, but they still appear in the same *order*.

Taking a second example with `s1 = abcdef` and `s2 = aacf`:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c}
  \textcolor{#81a1c1}{\texttt{\text{abcdef}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{b}} & \; 
  \textcolor{#bf616a}{\texttt{\text{c}}} & \; 
  \texttt{\text{d}} & \; 
  \texttt{\text{e}} & \; 
  \textcolor{#bf616a}{\texttt{\text{f}}} & \; 
\end{array}
$$

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c}
  \textcolor{#81a1c1}{\texttt{\text{aacf}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{a}} & \; 
  \textcolor{#bf616a}{\texttt{\text{c}}} & \; 
  \textcolor{#bf616a}{\texttt{\text{f}}} & \; 
\end{array}
$$

Here, the LCS is `acf`.

Hence, it can also be a sequence that is scattered across *both* strings.

The most efficient solution to this problem involves the use of *dynamic programming*, where we recursively compute the LCS of substrings
(more on this later).

$\textcolor{#b48ead}{\texttt{\text{fᵤₙ fₐcₜ (˙𐃷˙)}}}$ `diff` (and by extension, `git diff`) uses [LCS under the hood](https://en.wikipedia.org/wiki/Diff#Algorithm) 
to find the smallest set of changes between two files.

### Dynamic Programming Solution

Let's initialize a matrix with `m+1` rows and `n+1` columns where:
- `m` is the length of `s2`
- `n` is the length of `s1`
- first row and column are filled with `0`s

Taking `s1 = abcdef` and `s2 = aacf`, it is a `5x7` matrix.

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\texttt{\text{.}}} & \; \textcolor{#81a1c1}{\texttt{\text{a}}} & \; \textcolor{#81a1c1}{\texttt{\text{b}}} & \; \textcolor{#81a1c1}{\texttt{\text{c}}} & \; \textcolor{#81a1c1}{\texttt{\text{d}}} & \; \textcolor{#81a1c1}{\texttt{\text{e}}} & \; \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq
\end{array}
$$

Let's establish a few rules:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\texttt{\text{.}}} & \; \textcolor{#81a1c1}{\texttt{\text{a}}} & \; \textcolor{#81a1c1}{\texttt{\text{b}}} & \; \textcolor{#81a1c1}{\texttt{\text{c}}} & \; \textcolor{#81a1c1}{\texttt{\text{d}}} & \; \textcolor{#81a1c1}{\texttt{\text{e}}} & \; \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \; \colorbox{#bf616a}{\texttt{p}} & \; \colorbox{#bf616a}{\texttt{q}} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \colorbox{#bf616a}{\texttt{r}} & \; \colorbox{#bf616a}{\texttt{s}} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq
\end{array}
$$

- If the characters of the current square `s` match (in this case, `a` and `a`), then the value of `s` is `p + 1` (upper left diagonal + 1).
- If they don't match, we take $max(r, q)$.

$$
\text{s} =
\begin{cases}
\text{p} + 1            & \text{if } s_1[i] = s_2[j] \\
\max(\text{r}, \text{q}) & \text{if } s_1[i] \neq s_2[j]
\end{cases}
$$

Where $(i, j)$ are the coordinates of the current square, and `s1` and `s2` are the strings we defined.

Considering the first unfilled square (marked in red), the characters *do* match (`a` and `a`):

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\texttt{\text{.}}} & \; \textcolor{#81a1c1}{\texttt{\text{a}}} & \; \textcolor{#81a1c1}{\texttt{\text{b}}} & \; \textcolor{#81a1c1}{\texttt{\text{c}}} & \; \textcolor{#81a1c1}{\texttt{\text{d}}} & \; \textcolor{#81a1c1}{\texttt{\text{e}}} & \; \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \; \colorbox{#b48ead}{\texttt{0}} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq
\end{array}
$$

Hence, we write the value of this square as the value of the upper left diagonal square (marked in purple) + 1, which is `0 + 1 = 1`.

Considering the second unfilled square:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\texttt{\text{.}}} & \; \textcolor{#81a1c1}{\texttt{\text{a}}} & \; \textcolor{#81a1c1}{\texttt{\text{b}}} & \; \textcolor{#81a1c1}{\texttt{\text{c}}} & \; \textcolor{#81a1c1}{\texttt{\text{d}}} & \; \textcolor{#81a1c1}{\texttt{\text{e}}} & \; \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{1} & \; \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq
\end{array}
$$

The characters *do not* match (`a` and `b`). So we take the maximum value of the left square and the top square.
In this case, $max(1, 0)$, which is 1.

Filling out the rest of the row and moving to the second iteration:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\texttt{\text{.}}} & \; \textcolor{#81a1c1}{\texttt{\text{a}}} & \; \textcolor{#81a1c1}{\texttt{\text{b}}} & \; \textcolor{#81a1c1}{\texttt{\text{c}}} & \; \textcolor{#81a1c1}{\texttt{\text{d}}} & \; \textcolor{#81a1c1}{\texttt{\text{e}}} & \; \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \colorbox{#b48ead}{\texttt{0}} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq
\end{array}
$$

The characters match (`a` and `a`), so we put the value as `0 + 1 = 1`.

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\texttt{\text{.}}} & \; \textcolor{#81a1c1}{\texttt{\text{a}}} & \; \textcolor{#81a1c1}{\texttt{\text{b}}} & \; \textcolor{#81a1c1}{\texttt{\text{c}}} & \; \textcolor{#81a1c1}{\texttt{\text{d}}} & \; \textcolor{#81a1c1}{\texttt{\text{e}}} & \; \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{1} & \; \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \; \texttt{0} & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq & \; \sq
\end{array}
$$

Filling out the rest of the squares:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\texttt{\text{.}}} & \; \textcolor{#81a1c1}{\texttt{\text{a}}} & \; \textcolor{#81a1c1}{\texttt{\text{b}}} & \; \textcolor{#81a1c1}{\texttt{\text{c}}} & \; \textcolor{#81a1c1}{\texttt{\text{d}}} & \; \textcolor{#81a1c1}{\texttt{\text{e}}} & \; \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} & \; \texttt{0} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} & \; \texttt{1} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{1} & \; \texttt{1} & \; \texttt{2} & \; \texttt{2} & \; \texttt{2} & \; \texttt{2} \\[1em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \; \texttt{0} & \; \texttt{1} & \; \texttt{1} & \; \texttt{2} & \; \texttt{2} & \; \texttt{2} & \; \colorbox{#bf616a}{\texttt{3}}
\end{array}
$$

Now this value `3` is the length of the longest subsequence. (LCS in this case is `acf`).

In order to obtain the actual LCS, we can backtrack from the bottom right corner of the matrix (which holds the LCS length) 
   with the following rules:
   
- If characters match, then go back to the upper left diagonal square.
- If they don't match, then move in the direction of the larger value, either up or left.

`s1` and `s2` are our strings:
$$
s_1 = s_1[1]\,s_1[2]\cdots s_1[m]
\qquad
s_2 = s_2[1]\,s_2[2]\cdots s_2[n]
$$

This is what I meant when I said the LCS is computed recursively with the LCS of substrings:
$$
d_{i-1,\,j} = \mathrm{LCS}\bigl(
  s_1[1\ldots i-1],\ s_2[1\ldots j]
\bigr)
$$

We can define our movement with the following formula:

$$
\text{move}(i,j) = \left\{
\begin{array}{l c l}
  \nwarrow   & \text{if } s_1[i] = s_2[j]    & \\
  \uparrow   & \text{if } s_1[i] \neq s_2[j] & \text{and } \max(d_{i-1,j},\, d_{i,j-1}) = d_{i-1,j} \\
  \leftarrow   & \text{if } s_1[i] \neq s_2[j] & \text{and } \max(d_{i-1,j},\, d_{i,j-1}) = d_{i,j-1}
\end{array}
\right.
$$

Then we can obtain the following path:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c c c c c c c}
  \hspace{0.5em} & \textcolor{#81a1c1}{\texttt{\text{.}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{a}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{b}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{c}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{d}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{e}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{f}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{0} \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \colorbox{#bf616a}{\texttt{0}} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} \\[0.2em]
   &  & \mkern-10mu \nwarrow \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \texttt{0} & \mkern-10mu  \mkern-10mu & \colorbox{#b48ead}{\texttt{1}} & \mkern-10mu \leftarrow \mkern-10mu & \colorbox{#bf616a}{\texttt{1}} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu \nwarrow \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{c}}} \hspace{0.5em} & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \colorbox{#b48ead}{\texttt{2}} & \mkern-10mu \leftarrow \mkern-10mu & \colorbox{#bf616a}{\texttt{2}} & \mkern-10mu \leftarrow \mkern-10mu & \colorbox{#bf616a}{\texttt{2}} & \mkern-10mu  \mkern-10mu & \texttt{2} \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu \nwarrow \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{f}}} \hspace{0.5em} & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{1} & \mkern-10mu  \mkern-10mu & \texttt{2} & \mkern-10mu  \mkern-10mu & \texttt{2} & \mkern-10mu  \mkern-10mu & \texttt{2} & \mkern-10mu  \mkern-10mu & \colorbox{#b48ead}{\texttt{3}}
\end{array}
$$

In every purple marked square, we move to the upper left diagonal square, since the characters are equal (in the case of `3`, `f` = `f`).
Then we add the character to our string.

| **Character** 	| **Row** 	| **Column** 	|
|---	|:---:	|---	|
| f 	| 5 	| 7 	|
| c 	| 4 	| 4 	|
| a 	| 3 	| 2 	|

Reversing this, we get the string "acf" (the LCS).

I have referenced [this video](https://www.youtube.com/watch?v=4ClOkX0SWW4) for the explanation, the animation makes it very intuitive
to understand.

### LCS and Global Alignment

In the introduction, I mentioned *global alignment problems*. LCS is also a global alignment problem.

- To find the LCS, the algorithm evaluates the *entirety* of both strings. It looks for a subsequence
that spans the entire global length of the strings, even if that pattern is stretched out.
- It preserves the *order* of the entire string.

Also, one characteristic of LCS is that mismatches are treated the same way as gaps (there is no gap penalty).

$$\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c}
  \textcolor{#81a1c1}{\texttt{\text{plan}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{p}}} & \; 
  \texttt{\text{l}} & \; 
  \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{-}} & \; 
  \textcolor{#bf616a}{\texttt{\text{n}}} & \; 
  \texttt{\text{-}} \\[0.5em]
  \textcolor{#81a1c1}{\texttt{\text{paint}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{p}}} & \; 
  \texttt{\text{-}} & \; 
  \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{i}} & \; 
  \textcolor{#bf616a}{\texttt{\text{n}}} & \; 
  \texttt{\text{t}}
\end{array}$$

Example with "plan" and "paint", LCS is `pan`.

## Needleman-Wunsch

In bioinformatics, Needleman-Wunsch (NW) is an algorithm used to align protein or nucleotide sequences. Just like the LCS solution, NW is also a global 
alignment algorithm. In fact, the LCS solution is actually a simplified version of NW (this is why we use the suspiciously similar
alignment matrix).

Generally, NW is used when two sequences are similar and they need to be aligned to check how they have mutated.

If we use a plain text analogy, consider two strings that are *supposed* to be similar but aren't (like in the case of spelling
mistakes and such). If we want to see *how much* they have diverged, we use NW.

With NW, you can assign an alignment "score" to every possible alignment that can exist between two strings. It is an optimal algorithm,
so it produces the best possible alignment for the chosen scoring system.

The primary reason I'm covering NW here is because Smith-Waterman builds on top of it, and is tailored for local alignment problems
(like fuzzy finding).

Before we begin, I would like to introduce the following terms:

- Substitution score
- Gap penalty
- Alignment/scoring matrix
- Sub-alignment maximum score

### Substitution Score

Just like in the LCS solution, we define `a` and `b` as our strings, having lengths `m` and `n` respectively.

$$
a = a[1]\,a[2]\cdots a[m]
\qquad
b = b[1]\,b[2]\cdots b[n]
$$

Mathematically, we define the substitution score as:

$$
\text{sub}(a_i, b_j) := 
\begin{cases} 
    +2 & \text{if } a_i = b_j \\ 
    -1 & \text{if } a_i \neq b_j 
\end{cases} 
\qquad \text{and} \qquad \text{gap} := -1.
$$

This just means that for the $i$th character of `a` and $j$th character of `b`,
- If they match, add our *match* score (in this case, 2)
- If they don't match, add our *mismatch* score (in this case, -1)

### Gap Penalty and Scoring Matrix

Then we also define our *gap penalty* as -1.

Consider two strings, `a = plan` and `b = paint`.

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c}
  \textcolor{#81a1c1}{\texttt{\text{plan}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{p}}} & \; 
  \texttt{\text{l}} & \; 
  \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{-}} & \; 
  \textcolor{#bf616a}{\texttt{\text{n}}} & \; 
  \texttt{\text{-}} \\[0.5em]
  \textcolor{#81a1c1}{\texttt{\text{paint}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{p}}} & \; 
  \texttt{\text{-}} & \; 
  \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{i}} & \; 
  \textcolor{#bf616a}{\texttt{\text{n}}} & \; 
  \texttt{\text{t}} \\[0.5em]
  \hline
  & \; \textcolor{#a3be8c}{\texttt{\text{+2}}} & \; 
  \texttt{\text{-1}} & \; 
  \textcolor{#a3be8c}{\texttt{\text{+2}}} & \; 
  \texttt{\text{-1}} & \; 
  \textcolor{#a3be8c}{\texttt{\text{+2}}} & \; 
  \texttt{\text{-1}}
\end{array}
$$

For this alignment, the score would be `2 + -1 + 2 + -1 + 2 + -1 = 3`.

The alignment/scoring matrix looks exactly the same as the one we defined in the LCS solution (since it's just a simplified version of NW).

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c c c c c}
  \hspace{0.5em} & \textcolor{#81a1c1}{\texttt{\text{.}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{p}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{a}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{i}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{n}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{t}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{p}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{l}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{n}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq
\end{array}
$$

### Sub-Alignment Maximum Score

Think of sub-alignment maximum scores as the squares in the scoring matrix.

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c c c c c}
  \hspace{0.5em} & \textcolor{#81a1c1}{\texttt{\text{.}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{p}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{a}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{i}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{n}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{t}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{p}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \colorbox{#bf616a}{\texttt{p}} & \mkern-10mu  \mkern-10mu & \colorbox{#bf616a}{\texttt{q}} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu \searrow \mkern-10mu & \downarrow & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{l}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \colorbox{#bf616a}{\texttt{r}} & \mkern-10mu \rightarrow \mkern-10mu & \colorbox{#b48ead}{\texttt{s}} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{n}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq
\end{array}
$$

Consider a case where we know the values of `p`, `q` and `r`. We need to compute the value for `s`, which is defined as $S(i,j)$,

...with $i$ and $j$ being the coordinates of the square `s` (in this case, `i=2` and `j=2`). Then,

- `p` would be $S(i-1, j-1)$
- `q` would be $S(i-1, j)$
- `r` would be $S(i, j-1)$

Let's establish a few rules:
- If we move from `p` -> `s` (diagonally), then `s` = `p` + substitution score.
- If we move from `q` -> `s` (downwards), then `s` = `q` + gap penalty.
- If we move from `r` -> `s` (towards right), then `s` = `r` + gap penalty.

The path we choose will depend on which value is the highest. Hence, we can define the formula for sub-alignment maximum score like so:

$$
S(i,j) = \max \begin{cases} 
S(i-1, j) + \text{gap} \\ 
S(i, j-1) + \text{gap} \\ 
S(i-1, j-1) + \text{sub}(a_i, b_j) 
\end{cases}
$$

We set $S(0,0) = 0$ (an empty alignment is worth 0 points).

*Side note: we cannot move in any direction other than these three because that would result in using the same characters again.*

### Example Path

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c}
  \textcolor{#81a1c1}{\texttt{\text{plan}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{p}}} & \; 
  \texttt{\text{l}} & \; 
  \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{-}} & \; 
  \textcolor{#bf616a}{\texttt{\text{n}}} & \; 
  \texttt{\text{-}} \\[0.5em]
  \textcolor{#81a1c1}{\texttt{\text{paint}}} \hspace{0.5em}
  & \; \textcolor{#bf616a}{\texttt{\text{p}}} & \; 
  \texttt{\text{-}} & \; 
  \textcolor{#bf616a}{\texttt{\text{a}}} & \; 
  \texttt{\text{i}} & \; 
  \textcolor{#bf616a}{\texttt{\text{n}}} & \; 
  \texttt{\text{t}} \\[0.5em]
  \hline
  & \; \textcolor{#a3be8c}{\texttt{\text{+2}}} & \; 
  \texttt{\text{-1}} & \; 
  \textcolor{#a3be8c}{\texttt{\text{+2}}} & \; 
  \texttt{\text{-1}} & \; 
  \textcolor{#a3be8c}{\texttt{\text{+2}}} & \; 
  \texttt{\text{-1}}
\end{array}
$$

The path for this alignment would look like so:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c c c c c}
  \hspace{0.5em} & \textcolor{#81a1c1}{\texttt{\text{.}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{p}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{a}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{i}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{n}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{t}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \colorbox{#b48ead}{\texttt{0}} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu \searrow \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{p}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu \searrow \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{l}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu & \downarrow & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu \searrow \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{n}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \mkern-10mu \rightarrow \mkern-10mu & \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \mkern-10mu \rightarrow \mkern-10mu & \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}}
\end{array}
$$


| a[i] | b[j] | next move | type  | score delta      | seq A | seq B |
|-------|:-----:|-----------|-------|------------------|-------|-------|
| p     | p     | diagonal  | match | match (+2)       | p     | p     |
| a     | l     | down      | gap   | gap penalty (-1) | l     | -     |
| a     | a     | diagonal  | match | match (+2)       | a     | a     |
| i     | n     | right     | gap   | gap penalty (-1) | -     | i     |
| n     | n     | diagonal  | match | match (+2)       | n     | n     |
| t     | n     | right     | gap   | gap penalty (-1) | -     | t     |

This example doesn't have a case where we move diagonally AND the characters mismatch, but if that happens, the score delta would be 
the mismatch value.

I have referenced [this master's thesis by Oliver Boes](https://github.com/oboes/gotoh/blob/master/doc/thesis.pdf) and 
[this youtube video](https://www.youtube.com/watch?v=xbcpnItE3_4) for the explanation.

### Implementation

1. Defining the NW struct:

```go
type Nw struct {
	match    int
	mismatch int
	gap      int
	dp       [][]int
}
```
Where `match` and `mismatch` are the substitution score delta for matching and non-matching characters respectively, `gap` is the gap penalty,
and `dp` is the alignment/scoring matrix.

2. Filling the alignment matrix:

We define a method for `Nw` called `fillTable` which takes in two sequences `a` and `b`.

```go
func (nw *Nw) fillTable(a []byte, b []byte) {
    // (a) fill out first row and column

    // iterate through all rows and columns of alignment matrix other than the first ones
	for i := 1; i < len(a)+1; i++ {
		for j := 1; j < len(b)+1; j++ {
            // (b) compute S(i,j)
		}
	}
}
```

(a) Filling out the first row and column:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c c c c c}
  \hspace{0.5em} & \textcolor{#81a1c1}{\texttt{\text{.}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{p}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{a}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{i}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{n}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{t}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \colorbox{#bf616a}{\texttt{0}} & \mkern-10mu \rightarrow \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   & \downarrow & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{p}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{l}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{n}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq
\end{array}
$$

We start from $S(0,0)$ and can move either right/down. In either case, we add a score delta of our gap penalty (-1), so in both cases,
the value would be `0 + -1 = -1`

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c c c c c}
  \hspace{0.5em} & \textcolor{#81a1c1}{\texttt{\text{.}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{p}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{a}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{i}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{n}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{t}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \texttt{0} & \mkern-10mu  \mkern-10mu & \colorbox{#bf616a}{\texttt{-1}} & \mkern-10mu \rightarrow \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{p}}} \hspace{0.5em} & \colorbox{#bf616a}{\texttt{-1}} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   & \downarrow & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{l}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{n}}} \hspace{0.5em} & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq
\end{array}
$$

Moving down/right again, this time the value is going to be `-1 + -1 = -2`. Filling out the rest of the first row and column like this:

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c c c c c c c c}
  \hspace{0.5em} & \textcolor{#81a1c1}{\texttt{\text{.}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{p}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{a}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{i}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{n}}} & \mkern-10mu  \mkern-10mu & \textcolor{#81a1c1}{\texttt{\text{t}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\texttt{\text{.}}} \hspace{0.5em} & \texttt{0} & \mkern-10mu  \mkern-10mu & \texttt{-1} & \mkern-10mu  \mkern-10mu & \texttt{-2} & \mkern-10mu  \mkern-10mu & \texttt{-3} & \mkern-10mu  \mkern-10mu & \texttt{-4} & \mkern-10mu  \mkern-10mu & \texttt{-5} \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{p}}} \hspace{0.5em} & \texttt{-1} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{l}}} \hspace{0.5em} & \texttt{-2} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{a}}} \hspace{0.5em} & \texttt{-3} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq \\[0.2em]
   &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  & \mkern-10mu  \mkern-10mu &  \\[0.2em]
  \textcolor{#81a1c1}{\texttt{\text{n}}} \hspace{0.5em} & \texttt{-4} & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq & \mkern-10mu  \mkern-10mu & \sq
\end{array}
$$

We can easily do this by defining two `for` loops:

```go
func (nw *Nw) fillTable(a []byte, b []byte) {
    // for 1st column
	for i := 0; i < len(a)+1; i++ {
		nw.dp[i][0] = nw.gap * i
	}
    // for 1st row
	for j := 0; j < len(b)+1; j++ {
		nw.dp[0][j] = nw.gap * j
	}
    // ...
}
```

(b) Computing $S(i,j)$

```go
func (nw *Nw) fillTable(a []byte, b []byte) {
    // ...
	for i := 1; i < len(a)+1; i++ {
		for j := 1; j < len(b)+1; j++ {
			diagDelta := nw.mismatch
			if a[i-1] == b[j-1] {
				diagDelta = nw.match
			}
			fromDiagScore := nw.dp[i-1][j-1] + diagDelta
			fromTopScore := nw.dp[i-1][j] + nw.gap
			fromLeftScore := nw.dp[i][j-1] + nw.gap
			nw.dp[i][j] = max(fromDiagScore, fromTopScore, fromLeftScore)
		}
	}
}
```

This is fairly straightforward. Looking at the substitution score formula:

$$
\text{sub}(a_i, b_j) := 
\begin{cases} 
    +2 & \text{if } a_i = b_j \\ 
    -1 & \text{if } a_i \neq b_j 
\end{cases} 
\qquad \text{and} \qquad \text{gap} := -1.
$$

`diagDelta` = $sub(a_i, b_j)$

Then we calculate $S(i,j)$ by using `diagDelta` and our gap penalty.

$$
S(i,j) = \max \begin{cases} 
S(i-1, j) + \text{gap} \\ 
S(i, j-1) + \text{gap} \\ 
S(i-1, j-1) + \text{sub}(a_i, b_j) 
\end{cases}
$$

3. Traceback:

In order to actually get our traceback sequences, we need to backtrack from the bottom right corner of the scoring matrix.

We define another method, `traceback` which takes in `a` and `b` and outputs two sequences.

```go
func (nw *Nw) traceback(a []byte, b []byte) ([]byte, []byte) {
    // (a) initialize buffers and pos
    // (b) while both sequences have chars, check if chars match / determine path taken
    // (c) handling edge cases (string 1 chars running out before string 2 or vice versa)
    // (d) return trimmed bufA, bufB
}
```

(a) Initializing buffers and `pos`

```go
func (nw *Nw) traceback(a []byte, b []byte) ([]byte, []byte) {
    maxLen := len(a) + len(b)
    bufA := make([]byte, maxLen)
    bufB := make([]byte, maxLen)
    pos := maxLen - 1
    // ...
}
```

(b) Backtracking loop

```go
func (nw *Nw) traceback(a []byte, b []byte) ([]byte, []byte) {
    // ...
    i, j := len(a), len(b)

    for i > 0 && j > 0 {
        score := nw.mismatch

        if a[i-1] == b[j-1] {
            score = nw.match
        }

        switch nw.dp[i][j] {
            // handling diagonal, top and left
        }
    }
    // ...
}
```

`Diagonal`
```go
case nw.dp[i-1][j-1] + score:
    bufA[pos] = a[i-1]
    bufB[pos] = b[j-1]
    i--
    j--
    pos--
```

`Top`
```go
case nw.dp[i-1][j] + nw.gap:
    bufA[pos] = a[i-1]
    bufB[pos] = '_'
    i--
    pos--
```

`Left/default`
```go
default:
    bufA[pos] = '_'
    bufB[pos] = b[j-1]
    j--
    pos--
```

(c) Edge cases (string 1 characters run out before string 2 or vice versa):

```go
func (nw *Nw) traceback(a []byte, b []byte) ([]byte, []byte) {
    // ...
    for ; i > 0; i-- {
        bufA[pos] = a[i-1]
        bufB[pos] = '_'
        pos--
    }

    for ; j > 0; j-- {
        bufA[pos] = '_'
        bufB[pos] = b[j-1]
        pos--
}
```

And finally, returning the trimmed buffers:

```go
func (nw *Nw) traceback(a []byte, b []byte) ([]byte, []byte) {
    // ...
    return bufA[pos+1:], bufB[pos+1:]
}
```

## Smith-Waterman

### Affine Gaps

### Bit Vectors

### SIMD Acceleration

## Results

## References

- [Frizbee](https://github.com/saghen/frizbee)
- [Longest Common Subsequence Problem Visually Explained](https://www.youtube.com/watch?v=4ClOkX0SWW4)
- [How similar are two words? | Needleman–Wunsch #SoME4](https://www.youtube.com/watch?v=xbcpnItE3_4)
- [Tinyalign](https://github.com/0xMukesh/tinyalign/)

