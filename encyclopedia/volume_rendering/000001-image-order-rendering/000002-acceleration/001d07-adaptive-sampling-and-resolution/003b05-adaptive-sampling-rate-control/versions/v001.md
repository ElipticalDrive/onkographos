# Adaptive Sampling Rate Control

Also called: Viewport Adaptive Resolution, dt_scale control

## Overview
Adaptive Sampling Rate Control modulates the ray marching step size ($\Delta t$) via a uniform scalar parameter ($dt\_scale$) to balance rendering performance and visual fidelity. It differs from error-driven methods by using a user-defined or heuristic rate multiplier rather than computing local gradient or error metrics [S1, unofficial]. It sets apart from [Resolution Adaptation](3063-resolution-adaptation.md) by adjusting the sampling density along the ray rather than the pixel resolution on the screen.

## How it works
**Inputs and Outputs:**
*   Input: A sampling rate multiplier $S$ (or index mapping to $S$).
*   Output: A uniform float $dt\_scale$ passed to the rendering shader.

**Data Structures:**
*   `dt_scale`: A single-precision floating-point value representing the inverse of the sampling rate.

**Steps:**
1.  Determine the desired sampling rate $S$. In some implementations, this is derived from a user interface slider with discrete indices $i \in [0, 5]$ mapped to specific rates:
    *   $i=0 \rightarrow S=0.25$
    *   $i=1 \rightarrow S=0.5$
    *   $i=2 \rightarrow S=1.0$
    *   $i=3 \rightarrow S=2.0$
    *   $i=4 \rightarrow S=4.0$
    *   $i=5 \rightarrow S=8.0$ [S1, unofficial]
2.  Compute the step size control parameter $dt\_scale$ as the inverse of the sampling rate:
    $$ dt\_scale = \frac{1.0}{S} $$
3.  Pass $dt\_scale$ to the volume rendering shader as a uniform variable [S1, unofficial].
4.  The shader uses $dt\_scale$ to modulate the ray marching step size $\Delta t$. Typically, the base step size is multiplied by $dt\_scale$ or used as a divisor depending on the specific shader implementation, effectively scaling the distance between sample points [S1, unofficial].

**Parameters:**
*   Sampling Rate $S$: Discrete values $\{0.25, 0.5, 1.0, 2.0, 4.0, 8.0\}$ [S1, unofficial].
*   $dt\_scale$: Uniform float, $dt\_scale = 1/S$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If $S=0$, division by zero occurs; implementations must clamp $S$ to a minimum positive value or handle the index mapping to ensure $S \ge 0.25$ [S1, unofficial].
*   High sampling rates ($S > 1.0$) significantly increase the number of volume queries, potentially causing frame rate drops [S1, unofficial].

## Subtopics

### Dense Sampling for Multidimensional Transfer Functions
A rendering strategy requiring dense sampling of voxel data to ensure smooth boundaries when using multidimensional transfer functions, as neighboring voxels may differ in additional dimensions [S2]. It is chosen when using complex transfer functions (e.g., 3D or higher) where standard sampling causes aliasing at boundaries [S2]. It sets apart from basic adaptive control by mandating a high fixed density rather than dynamic adjustment, though it may be combined with interactive rate lowering [S2].

*   **Definition:** Requires dense sampling to prevent straying across high-opacity regions in multidimensional transfer function space [S2].
*   **Parameters:**
    *   Rendering Sample Rate: 3 to 6 samples per voxel for high-quality frozen renderings [S2].
    *   Interaction Sample Rate: Lowered during user interaction for performance, with rerendering at higher rates after actions complete [S2].
*   **Performance:** Frame updates can take nearly a second for a $256 \times 256 \times 128$ shaded and shadowed volume at 3-6 samples per voxel [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the sampling rate $S$. Doubling $S$ doubles the number of ray marching steps and volume texture lookups [S1, unofficial].
*   **Memory Complexity:** Constant, independent of $S$, as it only stores the uniform $dt\_scale$ [S1, unofficial].
*   **Measured Performance:** For a $256 \times 256 \times 128$ volume with multidimensional transfer functions, high sampling rates (3-6 samples/voxel) result in frame times of nearly 1 second [S2].
*   **Strengths:** Simple to implement; provides predictable control over quality vs. speed via a single scalar [S1, unofficial].
*   **Limitations:** Does not adapt to local scene complexity; uniform step size may waste samples in empty space or undersample in complex regions if $S$ is not chosen carefully [S1, unofficial].
*   **When to use:** When global quality control is preferred over local error minimization, or when implementing interactive exploration with fallback high-quality rendering [S2].

## Background
The concept of adjusting sampling rates for performance is discussed in the context of multidimensional transfer functions, where dense sampling is necessary for visual quality but costly [S2]. The specific implementation of `dt_scale` as a uniform float controlling ray step size is found in WebGL volume rendering tutorials [S1, unofficial].

## Sources
[S1] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S2] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
