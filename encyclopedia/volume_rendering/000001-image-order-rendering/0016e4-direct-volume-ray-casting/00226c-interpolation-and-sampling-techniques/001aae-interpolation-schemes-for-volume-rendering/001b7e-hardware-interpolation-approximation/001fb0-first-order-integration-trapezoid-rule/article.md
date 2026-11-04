# First Order Integration (Trapezoid Rule)
Also called: Linear Interpolation Approximation, Linear approximation, Linear interpolation integration, Standard pre-integrated volume rendering

## Overview
First Order Integration approximates the volume rendering integral along a ray segment by assuming the scalar field varies linearly between the segment's endpoints, effectively applying the trapezoid rule [S1, unofficial][S2]. This method improves upon zero-order (piecewise-constant) Riemann sums by capturing higher frequencies of the transfer function [S2]. It sets itself apart from its parent, Hardware Interpolation Approximation, by explicitly modeling the linear variation of the scalar field to enable pre-integration, whereas the parent scheme relies on hardware interpolation assumptions [S1, unofficial][S2].

## How it works
**Inputs:**
*   Transfer function $q(s)$ mapping scalar value $s$ to chromaticity and extinction coefficient $\kappa(s)$ [S2].
*   Ray segment defined by front sample scalar value $s_f$, back sample scalar value $s_b$, and segment length $l$ [S2].

**Outputs:**
*   Pre-computed integrals for chromaticity (color contribution) and transparency (opacity) for the segment [S2].

**Data Structures:**
*   A 3D lookup table (LUT) indexed by $(s_f, s_b, l)$ storing the pre-computed integrals [S2].

**Steps:**
1.  **Approximate Scalar Field:** Model the scalar field $s(x)$ within the ray segment of length $l$ as a linear interpolation between the front sample $s_f$ and the back sample $s_b$ [S2].
    $$ s(x) = s_f + \frac{s_b - s_f}{l} x, \quad x \in [0, l] $$
2.  **Compose with Transfer Function:** Combine the linearly interpolated scalar field with the transfer function to obtain the position-dependent extinction coefficient $\kappa(s(x))$ and color contribution density $q(s(x))$ [S2].
3.  **Pre-compute Integrals:** Calculate the integrals of the extinction coefficient and color contribution over the segment length $l$ [S2].
    *   Transparency $T$:
        $$ T \approx \exp\left(-\int_0^l \kappa(s(x)) \, dx\right) $$
    *   Color Contribution $C$:
        $$ C \approx \int_0^l q(s(x)) \, dx $$
    *   In zero-order integration, these are approximated as $T_i \approx e^{-\kappa(s_i)\Delta x}$ and $c_i \approx q(s_i)\Delta x$ using a single point value [S1, unofficial]. First-order integration replaces this with the integral over the linear variation [S2].
4.  **Store in LUT:** Store the resulting $T$ and $C$ values in the 3D lookup table indexed by $(s_f, s_b, l)$ [S2].

**Parameters:**
*   $s_f$: Scalar value at the front endpoint of the ray segment [S2].
*   $s_b$: Scalar value at the back endpoint of the ray segment [S2].
*   $l$: Length of the ray segment [S2].

**Edge Cases and Pitfalls:**
*   **Continuity:** The reconstructed signal is $C^0$ continuous but not $C^1$ continuous inside a dual voxel (between two sample points) [S2]. This lack of $C^1$ continuity is inherent to the linear approximation of a trilinearly interpolated field [S2].
*   **Comparison to Riemann Sum:** Unlike the Riemann sum which uses the function value at a single point (piecewise-constant), this method uses linear interpolation between discrete points [S1, unofficial].

## Subtopics

### [Second Order Pre-Integrated Volume Rendering](3388-second-order-pre-integrated-volume-rendering.md)
Uses quadratic polynomials to model scalar field curvature, reducing approximation error for high-frequency transfer functions compared to linear approaches.

## Operating characteristics
*   **Time Complexity:** Pre-computation is $O(N^3)$ where $N$ is the resolution of the LUT dimensions for $s_f, s_b, l$ [S2]. Runtime rendering is $O(1)$ per segment via LUT lookup [S2].
*   **Memory Complexity:** $O(N^3)$ for the 3D lookup table [S2].
*   **Strengths:** Captures high frequencies of the transfer function better than zero-order integration [S2].
*   **Limitations:** Approximation error remains significant compared to higher-order methods due to lack of $C^1$ continuity [S2].
*   **When to use:** When rendering quality needs to be improved over zero-order methods [S2].
*   **When not to use:** Not given in the sources.

## Background
First Order Integration, also known as the trapezoid rule in numerical integration, was proposed in the field of volume visualization to improve rendering quality by capturing high frequencies of the transfer function [S2]. It builds on the concept of pre-integration, where the integral of the transfer function composition is pre-computed [S2].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
