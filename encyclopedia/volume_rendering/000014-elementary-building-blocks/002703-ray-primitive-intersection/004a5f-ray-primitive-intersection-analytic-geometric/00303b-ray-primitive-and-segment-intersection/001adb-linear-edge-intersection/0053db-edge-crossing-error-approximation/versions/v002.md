# Edge-Crossing Error Approximation

Also called: Linear interpolation error estimation, Taylor series error approximation

## Overview
This method estimates the positional error introduced by linear interpolation when determining the intersection of a scalar field with an isovalue on a grid edge [S1]. It leverages Taylor series expansion to bound the difference between the true root and the linear approximation without solving a linear system [S1].

## How it works
**Inputs:** Scalar field values at grid vertices $x_i, x_{i+1}$, the isovalue $k$, and the linear crossing position $\bar{x}^*$ computed via [Linear Edge Intersection](868-linear-edge-intersection.md) [S1].
**Outputs:** The estimated edge-crossing error $e$ (or bound $e_b$) [S1].

**Data Structures:**
The method requires the first-order divided differences $U[i, i+1]$ and second-order divided differences $U[i-1, i+1]$ and $U[i, i+2]$ [S1].
*   $U[i] = f(x_i)$
*   $U[i, i+1] = \frac{U[i+1] - U[i]}{x_{i+1} - x_i}$ [S1]
*   $U[i-1, i+1] = \frac{U[i, i+1] - U[i-1, i]}{x_{i+1} - x_{i-1}}$ [S1]
*   $U[i, i+2] = \frac{U[i+1, i+2] - U[i, i+1]}{x_{i+2} - x_i}$ [S1]

**Steps:**
1.  Compute the divided differences $U[i, i+1]$, $U[i-1, i+1]$, and $U[i, i+2]$ using the scalar values at vertices $x_{i-1}, x_i, x_{i+1}, x_{i+2}$ [S1].
2.  Calculate the maximum second-order divided difference magnitude: $M = \max(|U[i-1, i+1]|, |U[i, i+2]|)$ [S1].
3.  Compute the tighter vertex approximation error estimate $e$ using the linear crossing position $\bar{x}^*$:
    $$ e = \frac{M}{U[i, i+1]} (\bar{x}^* - x_i)(\bar{x}^* - x_{i+1}) $$
    [S1]
4.  (Optional) Compute the standard bound approximation $e_b$ for a conservative upper bound:
    $$ e_b = \frac{M}{U[i, i+1]} (x_i - x_{i+1})^2 $$
    [S1]

**Formulas:**
The theoretical error bound is derived from the Taylor series remainder:
$$ |x^* - \bar{x}^*| \le \frac{\max_{\xi \in [x_i, x_{i+1}]} |f''(\xi)|}{2 U[i, i+1]} (x^* - x_i)(x^* - x_{i+1}) $$
where $x^*$ is the true root and $\xi$ is an unknown point in the interval [S1]. Since $f''(\xi)$ and $x^*$ are unknown, they are approximated as described in Steps 2-3 [S1].

**Parameters:**
*   Grid spacing: Implicit in $x_i$ coordinates [S1].
*   Typical performance: Error computation takes <1ms for the Tangle dataset, compared to seconds for isosurface comparison methods [S1].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $U[i, i+1] \approx 0$, the denominator becomes small, leading to large error estimates or numerical instability. This occurs when the scalar values at the edge endpoints are equal [S1].
*   **Overestimation:** The standard bound $e_b$ tends to significantly overestimate the error compared to the measured error [S1]. The vertex approximation $e$ is much tighter [S1].
*   **Hidden Features:** Linear interpolation may miss isosurface patches (hidden features) entirely. This method estimates error on existing crossings but does not detect missing features; see [Hidden Features Detection and Reconstruction](#hidden-features-detection-and-reconstruction) for that capability [S1].

## Subtopics

### Hidden Features Detection and Reconstruction
This variant detects and reconstructs isosurface patches (hidden features) not captured by standard linear interpolation by analyzing slopes on cell edges [S1].
*   **Definition:** Identifies cells where linear interpolation fails to capture topology changes due to opposite slopes surrounding a feature [S1].
*   **Detection Condition:** A hidden feature is detected in a cell if for any of its edges, two of the three slopes (divided differences) of neighboring edges have opposite signs. In 1D, this is indicated by $U[i-1, i+1] \cdot U[i+1, i+2] < 0$ [S1].
*   **Reconstruction:** The detected cell is subdivided into smaller subcells using tri-cubic Lagrange polynomial interpolation. Linear interpolation for refinement does not recover these features [S1]. The Marching Cubes (or Squares) algorithm is then applied to each subcell [S1].
*   **Pitfalls:** Cell refinements lead to cracks in the reconstructed isosurface where refined cells meet unrefined neighbors. Crack-free techniques from adaptive mesh refinement are required to resolve these mismatches [S1].

## Operating characteristics
**Time Complexity:** $O(1)$ per edge, as it involves a constant number of arithmetic operations and divided difference calculations [S1].
**Memory Complexity:** $O(1)$ auxiliary memory per edge, requiring storage of local scalar values and divided differences [S1].
**Performance:** The approximated error computation is computationally efficient, taking less than 1ms for the Tangle example, whereas measuring error via isosurface comparison takes a few seconds [S1].
**Strengths:** Provides quick insight into isosurface quality without additional sampling or solving linear systems [S1]. The vertex approximation $e$ is significantly closer to the measured error than the standard bound $e_b$ [S1].
**Limitations:** It only estimates error for existing crossings; it does not detect missing features (hidden features) on its own [S1]. The standard bound $e_b$ is a loose overestimate [S1].

## Background
The method originates from the analysis of interpolation errors in volume rendering and isosurface extraction [S1]. It builds on the observation that Taylor series expansion can estimate local function error [S1]. Previous work by Moller et al. used Taylor series for smooth filters and error estimation in volume data approximation [S1]. This specific application to edge-crossing error in Marching Cubes algorithms was proposed to visualize isosurface discrepancies and guide decisions on using higher-order interpolation or higher-resolution data [S1].

## Sources
[S1] Estimation and Visualization of Isosurface Uncertainty from Linear and High-Order Interpolation Methods, https://arxiv.org/abs/2409.00043v1, fetched 2026-10-26
