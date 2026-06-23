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
spell checkers (it's a [global alignment]() algorithm).

The funny part is that fuzzy searching (in tools like [fff](https://github.com/dmtrKovalenko/fff)) is also done by the same algorithm used for local sequence alignment in bioinformatics.
After checking out [frizbee](https://github.com/saghen/frizbee/#smith-waterman), I decided to write my own fuzzy searching tool implementing
the Smith-Waterman algorithm.

...but why?

Fuzzy finding is a tool we often take for granted (atleast in my workflow), whether it be sifting through directories, or opening files,
or even just searching through things in general. I wanted to know the concept behind it.

$$
\def\sq{\boxed{\phantom{\rule{0.7em}{0.7em}}}}
\begin{array}{r | c c c c}
  \hspace{0.5em} & \; \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{ }}} & \; \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{f}}} & \; \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{o}}} & \; \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{o}}} \\[0.5em] \hline \\[-0.5em]
  \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{s}}} \hspace{0.5em} & \; \htmlClass{jbmono}{0} & \; \htmlClass{jbmono}{0} & \; \htmlClass{jbmono}{0} & \; \htmlClass{jbmono}{0} \\[1em]
  \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{o}}} \hspace{0.5em} & \; \htmlClass{jbmono}{0} & \; \fcolorbox{#d8dee9}{#bf616a}{\phantom{\rule{0.7em}{0.7em}}} & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{m}}} \hspace{0.5em} & \; \htmlClass{jbmono}{0} & \; \sq & \; \sq & \; \sq \\[1em]
  \textcolor{#81a1c1}{\htmlClass{jbmono}{\text{e}}} \hspace{0.5em} & \; \htmlClass{jbmono}{0} & \; \sq & \; \sq & \; \sq
\end{array}
$$

## Longest Common Sub Sequence

### Needleman-Wunsch

### Smith-Waterman

## Gap Affinities

## Bit Vectors

## SIMD Acceleration
