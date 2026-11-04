# GPU-based Ray Traversal on Tetrahedral Meshes
Also called: GPU-based Ray Tracing on Tetrahedral Meshes, Garrity's method, Ray tracing through tetrahedral grid, Tetrahedral traversal, Weiler et al. method

## Overview
This method performs volume rendering on tetrahedral meshes by executing fast ray traversal concurrently for all pixels on a GPU [S2, unofficial]. It solves the problem of accurately integrating optical properties through unstructured volumetric data by traversing the mesh topology directly using neighbor pointers [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh defining the volume geometry, vertex positions, and adjacency information (neighbor pointers) [S2, unofficial]. Optical properties (density $\sigma$, color/emission $c$) are assumed constant within each tetrahedron [S1, S2, unofficial].
*   **Output:** A rendered image where each pixel color is the result of ray-marching through the tetrahedral cells [S2, unofficial].

**Data Structures:**
*   **Vertex Data:** Stored in texture memory to enable efficient random access from shader code [S2, unofficial].
*   **Adjacency Data:** Stored in texture memory, mapping each tetrahedron to its neighboring tetrahedra across shared faces [S2, unofficial].
*   **Ray State:** For each pixel, the current ray origin $p_0$ and the current tetrahedron index [S2, unofficial].

**Algorithm Steps:**
The traversal is executed in a single render pass, concurrently for all pixels [S2, unofficial]. For every ray, the procedure follows these steps:

1.  **Find Entry Point:** Determine the entry point $p_0$ of the ray into the convex hull of the volume [S2, unofficial].
2.  **Find Exit Point and Calculate Density:** Determine the exit point $p_1$ of the current tetrahedron along the ray. Calculate the density values $\sigma$ for the current cell [S2, unofficial].
3.  **Blend Fragment:** Blend the new fragment using the cell density and the distance between $p_0$ and $p_1$. This step typically involves a precomputed lookup table for blending [S2, unofficial].
4.  **Update Origin:** Use the exit point $p_1$ as the new entry point $p_0$ for the next iteration [S2, unofficial].
5.  **Traverse:** Follow the neighbor pointer from the current tetrahedron to the adjacent tetrahedron sharing the exit face. Return to Step 2 [S1, S2, unofficial].

**Formulas:**
The contribution $E_i$ of the $i$-th segment (bounded by ray distances $s_i$ and $s_{i+1}$, length $\Delta s_i = s_{i+1} - s_i$) to the final radiance $E$ is calculated by integrating the volume rendering integral [S1, unofficial]:

$$
E_i = \int_{s_i}^{s_{i+1}} \tau_i c_i T(s) ds
$$

Where:
*   $\tau_i$ is the extinction coefficient (density) within the $i$-th tetrahedron, assumed constant [S1, unofficial].
*   $c_i$ is the emitted color or source term within the $i$-th tetrahedron, assumed constant [S1, unofficial].
*   $T(s)$ is the transmissivity up to distance $s$, defined as $T(s) = \exp\left(-\int_0^s \tau(u) du\right)$ [S1, unofficial].

Substituting and separating the transparency up to $s_i$:

$$
E_i = \tau_i c_i \int_{s_i}^{s_{i+1}} \exp\left(-\int_0^{s_i} \tau(u) du\right) \cdot \exp\left(-\int_{s_i}^s \tau(u) du\right) ds
$$

Since $\tau(u) = \tau_i$ is constant within the segment:

$$
E_i = \tau_i c_i \exp\left(-\int_0^{s_i} \tau(u) du\right) \int_{s_i}^{s_{i+1}} \exp\left(-\tau_i (s - s_i)\right) ds
$$

$$
E_i = \tau_i c_i T(s_i) \left[ \frac{-1}{\tau_i} \exp\left(-\tau_i (s - s_i)\right) \right]_{s_i}^{s_{i+1}}
$$

$$
E_i = c_i T(s_i) \left( 1 - \exp\left(-\tau_i \Delta s_i\right) \right)
$$

The total color is the sum of contributions from all segments, weighted by the transmissivity of subsequent segments [S1, unofficial].

**Parameters:**
*   **Precomputed Lookup Table:** Used in Step 3 for blending. Assumes a static mesh [S2, unofficial].
*   **Imaginary Cells:** Added to support non-convex meshes and avoid re-entry [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Meshes:** The method requires a precomputed lookup table for blending, which assumes a static mesh. This is a critical drawback for dynamic volumes like smoke, where pre-calculation is infeasible [S2, unofficial].
*   **Completeness Issues:** The method suffers from completeness issues due to arbitrary distortion of the initial volume over time. It cannot guarantee correct fragment calculation and completeness when the volume deforms [S2, unofficial].
*   **Non-Convex Meshes:** While imaginary cells are added to avoid re-entry, the method still fails to handle local and global intersections correctly, leading to incomplete fragment lists [S2, unofficial].
*   **Memory Access:** Global random access memory is required for the implementation of a smoke renderer, as texture memory alone may not suffice for all data dependencies [S2, unofficial].

## Subtopics

### Local Intersection Handling in Ray Traversal
Local intersection (or self-intersection) occurs when a tetrahedron is flipped and cuts into another tetrahedron's volume [S2, unofficial]. Because the algorithm is not aware of local intersections and does not analyze the neighborhood, the intersecting tetrahedron fails to contribute to the viewing ray, resulting in an invalid pixel color due to an incomplete list of fragments [S2, unofficial].

## Operating characteristics
*   **Execution Model:** Ray traversal is executed in a single render pass, concurrently for all pixels on the GPU [S2, unofficial].
*   **Strengths:** Fast ray traversal executed on the graphics adapter; concurrent mesh traversal; supports non-convex meshes via imaginary cells [S2, unofficial].
*   **Limitations:** Inefficient for dynamic meshes due to reliance on precomputed lookup tables; completeness issues with deforming volumes; artifacts from incorrect handling of local/global intersections in non-convex meshes [S2, unofficial].
*   **When to Use:** Static tetrahedral volume rendering where high accuracy is required and dynamic deformation is not a factor [S2, unofficial].
*   **When Not to Use:** Dynamic smoke or fluid simulation where the mesh deforms over time [S2, unofficial].
*   **Complexity:** Not given in the sources.

## Background
The method is based on Garrity's algorithm for tracing rays through tetrahedral grids by determining the exit face and following neighbor pointers [S1, unofficial]. Weiler et al. implemented this approach on a GPU, storing vertex and adjacency data in texture memory to enable fast, concurrent mesh traversal [S1, S2, unofficial]. It is cited as a promising concept for volume-based rendering of tetrahedral data [S2, unofficial].

## Sources
[S1] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (unofficial)
[S2] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
