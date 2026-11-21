# Multi-pass shaded isosurfaces

Also called: Fast Shaded Isosurfaces, Shaded Isosurfaces via Multi-Pass, Westermann and Ertl

## Overview
Multi-pass shaded isosurfaces is a fast approach to display shaded isosurfaces by exploiting rasterization hardware, avoiding the performance loss associated with software-based shading or complex one-pass texture interpolation methods [S1, unofficial][S2, book]. It sets itself apart from Dachille et al.'s one-pass approach, which relies on 3D texture hardware interpolation combined with software shading and classification, by decomposing the shading computation into multiple rasterization passes [S1, unofficial][S2, book].

## How it works
The method is described as a "fast multi-pass approach" [S1, unofficial][S2, book]. However, the provided sources do not contain the specific algorithmic steps, data structures, formulas, or implementation details required to reconstruct the method from plain text. The sources only identify the authors (Westermann and Ertl) and the general category of the technique.

## Operating characteristics
Not given in the sources.

## Background
The method was introduced by Westermann and Ertl as a response to the significant loss in rendering performance caused by relying on rasterization hardware for certain shading tasks [S1, unofficial][S2, book]. It emerged during a period when advanced PC graphics hardware made texture-mapped volume rendering accessible to a broader community, driven by manufacturers like NVidia, ATI, and 3DLabs [S1, unofficial][S2, book]. It is cited alongside other approaches such as Dachille et al.'s one-pass method and octree-based subdivision methods by Boada et al. and LaMar et al. [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
