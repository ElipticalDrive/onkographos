# Gaussian Reconstruction Filter

Also called: Gaussian fit to Blackman-Harris 3.3, Gaussian Weighting Filter

## Overview
The Gaussian Reconstruction Filter is a spatial weighting function applied to jittered samples to improve edge quality and stability in volumetric rendering, specifically replacing box filters in froxel-based temporal anti-aliasing [S1, unofficial]. It is also used in A-buffer methods to compute data weights based on distance from a node [S2, unofficial]. It sets itself apart from its sibling, the [Gaussian Kernel Volumetric Primitive](4010-gaussian-kernel-volumetric-primitive.md), by acting as a post-sampling reconstruction filter for discrete jittered points rather than defining the primitive's shape itself.

## How it works
**Inputs:**
*   Sample positions relative to the filter center (frocel center or grid node).
*   Filter radius $r$ (for the variant in [S2, unofficial]).
*   Standard deviation multiplier $n$ (for the variant in [S2, unofficial]).
*   Radius multiplier $m$ (for the variant in [S2, unofficial]).

**Outputs:**
*   Normalized weight $W(x)$ or $w$ for each sample.

**Data Structures:**
*   Not specified. The filter operates on scalar distances.

**Steps:**
1.  Calculate the distance $x$ (or $d$) from the sample to the center of the froxel (or node).
2.  Compute the weight using the appropriate formula based on the specific variant being implemented.
3.  Apply the weight to the sample's contribution during accumulation.

**Formulas:**

*Variant 1: Gaussian fit to Blackman-Harris 3.3 [S1, unofficial]*
$$W(x) = e^{-2.29x^2}$$
Where:
*   $W(x)$ is the sample weight.
*   $x$ is the distance from the center of the froxel.

*Variant 2: Gaussian Weighting Filter [S2, unofficial]*
First, calculate the variance term:
$$2\sigma^2 = 2 \left(\frac{r \cdot m}{n}\right)^2$$
Then, compute the weight:
$$w = e^{-\frac{d^2}{2\sigma^2}}$$
Where:
*   $w$ is the weight.
*   $d$ is the Euclidean distance from the node to the data point.
*   $r$ is the filter radius.
*   $m$ is the radius multiplier.
*   $n$ is the number of standard deviations the radius represents (standard deviation multiplier).

**Parameters:**
*   **Blackman-Harris variant:** The coefficient $2.29$ is fixed [S1, unofficial].
*   **Gaussian Weighting variant:**
    *   $n$ (standard deviation multiplier): User-specified number of standard deviations the filter radius represents [S2, unofficial].
    *   $r$ (filter radius): Defines the spatial extent [S2, unofficial].
    *   $m$ (radius multiplier): Used in variance calculation [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Normalization:** In the [S2, unofficial] variant, the weight is normalized to the filter radius to guarantee values between 0 and 1, preventing arithmetic overflow or underflow [S2, unofficial].
*   **Distance Calculation:** $d$ must be the Euclidean distance [S2, unofficial].
*   **Degenerate Input:** If $n=0$, the variance calculation in [S2, unofficial] would be undefined; however, $n$ represents a user-specified multiplier for standard deviations, implying $n > 0$ [S2, unofficial].

## Subtopics

### Gaussian Weighting Filter
A filtering function for the A-buffer method that computes weights based on the squared distance error or Gaussian distribution [S2, unofficial]. It differs from the Blackman-Harris fit by allowing user control over the falloff shape via the standard deviation multiplier $n$ [S2, unofficial]. Choose this when explicit control over the Gaussian spread relative to a defined filter radius is required, or when implementing A-buffer density weighting [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Time: $O(1)$ per sample for the Blackman-Harris variant [S1, unofficial]. $O(1)$ per sample for the Gaussian Weighting variant, plus $O(1)$ for variance calculation [S2, unofficial].
*   Memory: $O(1)$ auxiliary storage [S1, S2, unofficial].

**Measured Performance:**
*   The Blackman-Harris fit provides better edge quality and stability under motion compared to a spatiotemporal box filter [S1, unofficial].
*   The Gaussian Weighting Filter avoids arithmetic overflow/underflow through normalization [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Improved edge quality over box filters [S1, unofficial]; stable under motion [S1, unofficial]; prevents overflow/underflow [S2, unofficial].
*   **Limitations:** Requires distance calculation [S1, S2, unofficial]. The [S2, unofficial] variant requires user tuning of $n$ and $m$ [S2, unofficial].

**When to Use:**
*   Use the Blackman-Harris variant for froxel-based TAA to replace box filters [S1, unofficial].
*   Use the Gaussian Weighting variant for A-buffer methods where parabolic or Gaussian falloff is desired [S2, unofficial].

## Background
The Gaussian fit to Blackman-Harris 3.3 was introduced by Karis in the context of temporal anti-aliasing for real-time froxel-based volumetric rendering to address the poor edge quality and instability of box filters [S1, unofficial]. The Gaussian Weighting Filter was developed for sampling scattered data onto rectangular grids for volume visualization, offering an alternative to density weighting based on squared distance error [S2, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S2] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
