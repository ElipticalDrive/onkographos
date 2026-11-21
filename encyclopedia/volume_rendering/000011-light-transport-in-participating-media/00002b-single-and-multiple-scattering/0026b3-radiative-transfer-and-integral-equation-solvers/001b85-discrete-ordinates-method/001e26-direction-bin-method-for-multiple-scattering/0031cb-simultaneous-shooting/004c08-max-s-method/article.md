# Max's Method
Also called: Global Interactions (Max)

## Overview
Max's Method is a volume rendering technique for participating media that approximates radiative transfer by propagating energy across a grid, thereby avoiding the explicit computation of $M^2 v^2$ form factors required by finite element formulations [S1][S2, book]. It builds on [Simultaneous Shooting](6602-simultaneous-shooting.md) and distinguishes itself from sibling methods by distributing flux to neighbor voxels simultaneously for all voxels in a layer and accumulating attenuation along a set of possible propagation paths rather than a single straight line, which reduces the "ray effect" [S1][S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid representing a participating medium (e.g., clouds or underwater scenes) with defined scattering and absorption properties, discretized into $M$ direction bins [S1][S2, book][S3, unofficial].
*   **Output:** The converged radiance or flux distribution within the volume [S2, book].

**Data Structures:**
*   **Voxel Grid:** A 3D array where each voxel stores flux values. Exact field layouts and bit-packing are Not given in the sources.
*   **Direction Bins:** The angular domain is discretized into $M$ bins [S1][S2, book].
*   **Layers:** Voxels are grouped into layers (slices) to enable simultaneous propagation [S1][S2, book].

**Algorithm Steps:**
1.  **Initialization:** Initialize flux values in all voxels based on source terms (e.g., light sources or initial emission) [S2, book].
2.  **Iteration Loop:** Repeat until convergence or a fixed iteration count:
    1.  **Layer Processing:** For each layer of voxels in the grid:
        1.  **Flux Distribution:** For each direction bin, distribute the flux from the current voxel to related neighbor voxels simultaneously for all voxels in the layer [S1][S2, book].
        2.  **Attenuation Accumulation:** Accumulate attenuation between voxels along a set of possible propagation paths, rather than along the straight line joining them [S1][S2, book].
        3.  **Multiple Scattering:** Account for multiple scattering events produced within a single receiving element [S1][S2, book].
    2.  **Scattering to Bins:** Scatter the received flux of each voxel to the direction bins [S1][S2, book].
3.  **Visualization Pass:** Compute pixel radiances using the source radiances of the voxels [S2, book].

**Formulas and Parameters:**
*   Let $v$ be the number of voxels in a layer or related dimension, $M$ be the number of direction bins, and $n$ be a parameter related to grid size or interaction count [S1].
*   The time to scatter the received flux of a voxel to direction bins is $O(Mv)$ [S1].
*   The propagation cost for $v^2$ interactions is $O(v \log n)$ [S1].
*   The final cost per iteration is $O(Mv \log n + M^2 v)$ [S1].
*   Source [S2, book] reports the scatter time as $O(1)$ and the final cost per iteration as $O(N)$ where $N$ is the total number of voxels, noting this avoids the $O(N^2)$ cost of computing all form factors [S2, book]. The sources differ in complexity notation; [S1] uses $M$ and $v$, while [S2, book] uses $N$ [S1][S2, book].

**Edge Cases and Pitfalls:**
*   **Ray Effect:** The method reduces the "ray effect" because energy is propagated through the whole bin, not only through a single direction [S1][S2, book].
*   **Convergence:** The method is most efficient when the number of iterations required to converge is small compared to $v$ [S1]. If many iterations are needed, computing the whole set of form factors (cost $O(v^2 n + v^2 M)$) and solving the resulting system may be preferable [S1].

## Operating characteristics
*   **Time Complexity:** The final cost per iteration is $O(Mv \log n + M^2 v)$ according to [S1], or $O(N)$ according to [S2, book]. This is better than computing the whole set of form factors (cost $O(v^2 n + v^2 M)$ or $O(N^2)$) when the number of iterations required to converge is small compared to $v$ [S1][S2, book].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Avoids the high cost of computing $M^2 v^2$ form factors; reduces ray effects; efficient for scenes requiring few scattering iterations [S1][S2, book].
*   **Limitations:** Efficiency degrades if many iterations are required; approximation of attenuation paths may lack precision compared to exact ray tracing [S1].
*   **When to use:** Rendering clouds or participating media where computational efficiency is prioritized and the scattering order is low [S1][S2, book].
*   **When not to use:** When high precision in attenuation paths is required or when the medium requires many scattering iterations to converge [S1].

## Background
Max's Method, also known as Global Interactions, was proposed by Max [S1][S2, book][S3, unofficial]. It is devoted to rendering clouds and has been applied to underwater scenes where the sea is treated as a participating medium characterized by real experimental medium parametrizations [S1]. It falls under the category of methods that consider a set of specific directions, which can be considered a restriction of the 3D-filter used in other methods [S1][S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
[S3] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
