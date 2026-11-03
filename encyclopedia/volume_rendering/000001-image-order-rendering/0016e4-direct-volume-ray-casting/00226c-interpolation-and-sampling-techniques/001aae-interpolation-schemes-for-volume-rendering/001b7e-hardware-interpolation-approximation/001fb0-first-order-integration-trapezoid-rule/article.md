# First Order Integration (Trapezoid Rule)

Also called: Linear Interpolation Approximation, Linear approximation, Linear interpolation integration, Standard pre-integrated volume rendering

## Overview
First Order Integration approximates the volume rendering integral by assuming a linear interpolation of the scalar field between two discrete sample points, corresponding to the trapezoid rule [S1, S2]. Unlike the piecewise-constant Riemann sum used in zero-order integration, this method captures higher frequencies of the transfer function, reducing visual artifacts caused by discontinuities between consecutive samples [S1, S2]. It sets itself apart by requiring pre-computation of integrals into a lookup table indexed by segment endpoints and length, rather than evaluating the transfer function per sample [S2].

## How it works
**Inputs:**
*   Scalar field values at the front endpoint ($s_f$) and back endpoint ($s_b$) of a ray segment [S2].
*   Length of the ray segment ($l$) [S2].
*   Transfer function mapping scalar values to chromaticity ($q$) and extinction coefficient ($\kappa$) [S2].

**Outputs:**
*   Pre-computed transparency ($T$) and color contribution ($c$) for the segment [S2].

**Data Structures:**
*   **3D Lookup Table (LUT):** A 3D array indexed by $(s_f, s_b, l)$ [S2].
    *   Dimensions: Determined by the quantization of $s_f$, $s_b$, and $l$ [G].
    *   Content: Each entry stores the pre-integrated transparency and color contribution for the segment defined by those parameters [S2].

**Steps:**
1.  **Define Linear Interpolation:** Approximate the scalar field $s(x)$ within a ray segment of length $l$ as a linear interpolation between the front sample $s_f$ and back sample $s_b$ [S2]. Let $x$ be the distance from the front endpoint ($0 \le x \le l$). The scalar value at position $x$ is:
    $$ s(x) = s_f + \frac{s_b - s_f}{l} x $$
2.  **Compose with Transfer Function:** The extinction coefficient $\kappa(x)$ and chromaticity $q(x)$ are obtained by applying the transfer function to the interpolated scalar field [S2]:
    $$ \kappa(x) = \kappa(s(x)) $$
    $$ q(x) = q(s(x)) $$
3.  **Integrate over Segment:** Compute the transparency $T$ and color contribution $c$ for the segment by integrating the composed functions over the segment length $l$ [S2]:
    $$ T = \exp\left( -\int_0^l \kappa(s(x)) \, dx \right) $$
    $$ c = \int_0^l q(s(x)) \exp\left( -\int_0^x \kappa(s(u)) \, du \right) \, dx $$
    These integrals are computed numerically during pre-processing [S2].
4.  **Store in LUT:** Store the computed $T$ and $c$ in the 3D lookup table at the index corresponding to $(s_f, s_b, l)$ [S2].

**Parameters:**
*   $s_f$: Scalar value at the front endpoint of the ray segment [S2].
*   $s_b$: Scalar value at the back endpoint of the ray segment [S2].
*   $l$: Length of the ray segment [S2].

**Edge Cases and Pitfalls:**
*   **Continuity:** The reconstructed signal is $C^0$ continuous globally but not $C^1$ continuous inside a dual voxel (between two sample points) [S2].
*   **Pre-computation Requirement:** Unlike zero-order integration, first-order integration requires pre-integration because the scalar field is not constant [S2].
*   **Sampling Distance:** The sampling distance $l$ affects the computation of color and opacity values, making it a necessary index for the LUT [S2].

## Operating characteristics
*   **Time Complexity:** Pre-computation is performed offline [S2]. Runtime rendering involves 3D texture lookups [S2].
*   **Memory Complexity:** Requires a 3D lookup table indexed by $(s_f, s_b, l)$ [S2]. The size depends on the resolution of each dimension [S2].
*   **Performance:** Better convergence than zero-order approximation, capturing higher frequencies of the transfer function [S2].
*   **Strengths:** Reduces visual artifacts from discontinuities between samples; captures high frequencies of the transfer function [S2].
*   **Limitations:** Requires pre-computation; not $C^1$ continuous [S2].
*   **When to use:** When higher visual quality is needed than zero-order integration provides [S2].
*   **When not to use:** Not given in the sources.

## Background
First Order Integration, also known as the trapezoid rule approximation in volume rendering, was proposed to improve rendering quality by capturing high frequencies of the transfer function [S2]. It addresses the approximation error and visual artifacts associated with zero-order (piecewise-constant) integration by using linear interpolation between samples [S1, S2]. The method builds on the concept of pre-integrated volume rendering, where integrals are pre-computed and stored in lookup tables [S2].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
