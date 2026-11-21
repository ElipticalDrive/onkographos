# Shape Space Scaling

Also called: Antisymmetric Mapping, Antisymmetric Mapping for Tensor Glyphs, POS mapping, Sign-preserving mappings

## Overview
Shape Space Scaling maps tensor invariants (eigenvalues) to a scale suitable for visualization, addressing the challenge that indefinite tensors have positive, negative, or zero eigenvalues while many visualization methods require positive values [S1, book]. It provides a positive-definite metric for glyph sizing that distinguishes between positive and negative eigenvalues [S1, book]. It directly transforms eigenvalue magnitudes for glyph sizing and spatial domain rendering, operating differently from sibling methods like the [Mohr Diagram](8899-mohr-diagram.md) [S1, book].

## How it works
**Inputs and Outputs**
- Input: Tensor eigenvalues $\bar{\sigma}_i$ (where $i$ indexes the three principal values) [S1, book].
- Output: Scaled values $f(\bar{\sigma}_i)$ suitable for positive-definite metrics or sign-preserving visualization [S1, book].

**Data Structures**
The method operates on scalar eigenvalues. Eigenvectors are already normalized and do not require scaling or mapping [S1, book].

**Steps**
1. Determine the visualization requirement: whether the downstream method requires strictly positive values (POS) or can handle signed values (SP) [S1, book].
2. If using Sign-preserving mappings (SP), apply a linear, logarithmic, or histogram equalization mapping that preserves the sign of the input eigenvalues [S1, book].
3. If using Mapping to R+ (POS), apply the antisymmetric mapping function to convert eigenvalues to the positive domain while maintaining the distinction between positive and negative inputs [S1, book].
4. Apply a global scale parameter to adapt the final glyph size or visual intensity [S1, book].

**Formulas**
For the POS mapping (Antisymmetric Mapping), the function is defined as [S1, book]:

$$f(\bar{\sigma}_i, c, \alpha) = \exp(\alpha \arctan(c \cdot \bar{\sigma}_i))$$

Where:
- $\bar{\sigma}_i$ is the $i$-th eigenvalue of the tensor [S1, book].
- $c$ is a parameter determining the slope of the function in the origin [S1, book].
- $\alpha$ is a scaling parameter for the antisymmetric mapping function [S1, book].

**Parameters**
- $c$: Determines the slope of the antisymmetric mapping function in the origin. Typical values are not specified in the sources [S1, book].
- $\alpha$: A scaling parameter for the antisymmetric mapping function. Typical values are not specified in the sources [S1, book].
- Global scale parameter: Used to adapt the glyph's size. Value depends on the desired visual output [S1, book].

**Pitfalls**
- Eigenvectors do not require scaling as they are already normalized [S1, book].
- The choice of mapping (SP vs POS) depends heavily on the specific dataset and the desired visualization outcome; an optimal mapping is not universal [S1, book].
- Indefinite tensors pose a challenge because their sign reveals important physical characteristics, which may be lost if mapped to R+ without care [S1, book].

## Operating characteristics
Not given in the sources.

## Background
Shape Space Scaling is discussed in the context of visual analysis of stress tensor fields, specifically within the framework of Shape Space (Tensor Eigenvalue Space) visualization [S1, book]. It was proposed to handle the scaling of coordinate axes in shape space diagrams and spatial rendering where tensor invariants define color, transparency, glyph shape, and glyph size [S1, book]. The method references prior work on antisymmetric mappings [23, 29] within the source text [S1, book].

## Sources
[S1] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
