# Affine Arithmetic Filtering

Also called: Affine Arithmetic for Shader Averaging, Heidrich et al. filtering

## Overview
Affine Arithmetic Filtering is a method to obtain an average value of a procedural shader with an error bound over a finite area using affine arithmetic [S1, S2]. It sets itself apart from sibling methods like [Prefiltering](2928-prefiltering.md) by providing a rigorous mathematical error bound derived from the propagation of uncertainty, rather than relying on statistical variance estimates or sampling [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A procedural shader function $f(\mathbf{x})$ where $\mathbf{x}$ is the spatial coordinate; the finite area (e.g., pixel footprint) over which to average.
*   **Output:** The average value of the shader over the area, along with an error bound $\delta$ representing the loss in noise variance due to filtering [S1, S2].

**Data Structures:**
Affine arithmetic represents a value $x$ as a linear form:
$$ x = x_0 + \sum_{i=1}^{n} x_i \epsilon_i $$
where $x_0$ is the central value, $x_i$ are coefficients, and $\epsilon_i$ are independent noise symbols taking values in $[-1, 1]$ [S1, S2].

**Steps:**
1.  Represent the input coordinates of the shader as affine forms, capturing the uncertainty of the position within the pixel area [S1, S2].
2.  Evaluate the procedural shader using affine arithmetic operations. This propagates the uncertainty through the shader's mathematical operations [S1, S2].
3.  Extract the central value of the resulting affine form as the average shader value [S1, S2].
4.  Compute the error bound $\delta$ from the sum of the absolute values of the non-central coefficients of the resulting affine form [S1, S2]. This bound quantifies the loss in noise variance due to filtering [S1, S2].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths and Limitations:** Provides a rigorous error bound for procedural shader averaging [S1, S2].
*   **When to use:** When filtering procedural textures and an analytical expression of the noise variance is available [S1, S2].
*   **When not:** Not given in the sources.

## Background
Affine Arithmetic Filtering was presented by Heidrich et al. in [HSS98] [S1, S2]. It is part of the broader investigation into general methods for filtering procedural textures [S1, S2].

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
