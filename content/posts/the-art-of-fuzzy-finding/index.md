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
spell checkers (it is a [global alignment]() algorithm).

The funny part is that fuzzy searching (in tools like [fff](https://github.com/dmtrKovalenko/fff)) is also done by the same algorithm used for local sequence alignment in bioinformatics.
After checking out [frizbee](https://github.com/saghen/frizbee/#smith-waterman), I decided to write my own fuzzy searching tool implementing
the Smith-Waterman algorithm.

...but why?

Fuzzy finding is a tool we often take for granted (atleast in my workflow), whether it be sifting through directories, or opening files,
or even just searching through things in general. I wanted to know the concept behind it.


## Longest Common Subsequence

Let's consider a simpler problem space.

Consider two strings, `s1 = "ccdegf"` and `s2 = "def"`.

We need to find the longest sequence of characters that appear in the same string, but not necessarily adjacent to each other.

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

In the first string, *all* the characters are not adjacent to each other, but they still appear in the same *order*.

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

### Dynamic Programming Solution

Let's initialize a matrix with `m+1` rows and `n+1` columns where:
- `m` is the length of `s2`
- `n` is the length of `s1`
- first row and column is filled with `0`s

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

- If the characters of current square `s` match (in this case, `a` and `a`), then the value of `s` is `p + 1` (upper left diagonal + 1).
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

The characters *do not* match, (`a` and `b`). So we take the maximum value of left square and top square.
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

$$
s_1 = s_1[1]\,s_1[2]\cdots s_1[m]
\qquad
s_2 = s_2[1]\,s_2[2]\cdots s_2[n]
$$

$$
d_{i-1,\,j} = \mathrm{LCS}\bigl(
  s_1[1\ldots i-1],\ s_2[1\ldots j]
\bigr)
$$

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

## Needleman-Wunsch

## Smith-Waterman

### Gap Affinities

### Bit Vectors

### SIMD Acceleration

## Results

## References

- https://www.youtube.com/watch?v=4ClOkX0SWW4
