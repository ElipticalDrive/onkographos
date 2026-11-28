# Multi-pass shaded isosurfaces

Also called: Shaded Isosurfaces via Multi-Pass, Westermann and Ertl

## Overview
Multi-pass shaded isosurfaces is a fast approach to display shaded isosurfaces introduced by Westermann and Ertl [S1, unofficial][S2, book]. It builds on [Fast Shaded Isosurfaces](2105-fast-shaded-isosurfaces.md) and is distinguished from Dachille et al.'s one-pass method, which combines 3D texture hardware interpolation with software shading and classification, by utilizing multiple rasterization passes [S1, unofficial][S2, book].

## How it works
Inputs, outputs, data structures, algorithmic steps, formulas, parameters, and edge cases are not given in the sources [S1, unofficial][S2, book]. Pseudocode is not provided in the sources [S1, unofficial][S2, book].

## Operating characteristics
Not given in the sources.

## Background
Westermann and Ertl introduced the method alongside other volume rendering techniques that emerged as advanced PC graphics hardware made texture-mapped volume rendering more accessible [S1, unofficial][S2, book]. The sources note that relying on rasterization hardware previously led to significant rendering performance loss, and contrast this approach with Dachille et al.'s one-pass technique and octree-based subdivision methods by Boada et al. and LaMar et al. [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
