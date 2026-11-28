# Soft Shadows via Ray Tracing

Also called: Packet-based Soft Shadow Ray Tracing

## Overview
This method generates soft shadows from area light sources by tracing coherent packets of shadow rays from primary ray hit points to samples on the light source [S1]. Unlike rasterization-based impostor approaches, it integrates global illumination effects easily due to its ray tracing basis [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Primary ray hit points, area light source geometry, and a user-defined number of shadow rays per hit point [S1].
*   **Output:** Visibility information for soft shadow computation [G].

**Data Structures:**
*   **Ray Packet:** A structure containing $N$ rays sharing a common origin $P$ [G]. Layout: `origin` (3 floats), `directions` (array of $N$ vectors, each 3 floats) [G].

**Steps:**
1.  **Generate Shadow Ray Packets:** For each primary ray hit point $P$, generate $N$ shadow rays connecting $P$ to samples on the area light source [S1]. Each ray $i$ has direction $\vec{d}_i = (L_i - P) / \|L_i - P\|$, where $L_i$ is the $i$-th light sample position [G].
2.  **Traverse Shadow Rays:** Traverse the generated shadow rays identically to primary ray packets, leveraging their common origin for coherent traversal [S1].
3.  **Evaluate Occlusion:** For each ray in the packet, test for intersections with scene geometry before the light sample [G].
4.  **Compute Shadow Factor:** Calculate the ratio of unoccluded rays to $N$ to determine the visibility at $P$ [G].

**Parameters:**
*   **Number of shadow rays ($N$):** Controls the quality of the soft shadow. Higher $N$ yields smoother shadows but increases computational cost. Controlled interactively by the user [S1].
*   **Light Source Size:** The spatial extent of the area light source. Larger sources produce softer shadows. Controlled interactively by the user [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** When $P$ is very close to the light source, numerical instability may occur in direction calculation [G].
*   **Degenerate Input:** If the light source has no samples or $N=0$, the method fails [G].

## Subtopics

### [Soft Shadows via Ray Packets](5179-soft-shadows-via-ray-packets.md)
Generates soft shadows from area light sources by organizing shadow rays into coherent packets for optimized traversal. It differs from independent shadow ray casting by maintaining spatial coherence across rays sharing a common origin, which improves cache utilization during scene traversal. Choose this variant when traversal efficiency is critical and the underlying renderer supports coherent ray packet processing.

## Operating characteristics
*   **Performance Trade-offs:** The number of shadow rays and light source size are controlled interactively to balance quality and performance [S1].
*   **Time Complexity:** $O(N \cdot T)$ per hit point, where $N$ is the ray count and $T$ is traversal cost [G]. Coherent packet traversal may reduce $T$ via cache efficiency [G].
*   **Memory Complexity:** $O(N)$ per hit point to store the ray packet [G].
*   **Strengths:** Easy integration of global effects compared to rasterization; smooth shadow transitions; interactive quality/performance control [S1, G].
*   **Limitations:** Computational cost scales with $N$; requires ray tracing infrastructure [G].
*   **When to use:** When soft shadows from area lights are needed and ray tracing is available [G].
*   **When not to use:** When hard shadows suffice or strict real-time constraints limit $N$ [G].

## Background
This method was introduced in the context of coherent grid traversal for volume rendering, specifically for visualizing particle datasets [S1]. It leverages ray tracing to handle complex lighting effects that are difficult to implement in rasterization systems using impostor-based geometry [S1].

## Sources
[S1] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
