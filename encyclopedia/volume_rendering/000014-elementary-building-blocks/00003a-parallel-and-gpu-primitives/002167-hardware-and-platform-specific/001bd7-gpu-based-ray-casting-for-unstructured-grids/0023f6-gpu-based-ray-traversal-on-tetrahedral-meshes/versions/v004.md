# GPU-based Ray Traversal on Tetrahedral Meshes

Also called: GPU-based Ray Tracing on Tetrahedral Meshes, Garrity's method, Ray tracing through tetrahedral grid, Tetrahedral traversal, Weiler et al. GPU Ray Casting, Weiler et al. method

## Overview
This method implements volume rendering on tetrahedral meshes by traversing a ray through the mesh structure on the GPU, determining the exit face of the current tetrahedron and following neighbor pointers to adjacent cells [S1, book][S4, unofficial]. It distinguishes itself from CPU-based traversal by storing all mesh and traversal data in GPU texture memory, enabling concurrent, single-pass traversal for all pixels without CPU-GPU data transfer [S2, S3, unofficial]. Unlike sibling methods that may use hierarchical structures or voxelization, this approach relies on explicit tetrahedral adjacency and convexity assumptions [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh (vertices, connectivity, density/scalar field), camera parameters (ray origin and direction per pixel), and optical properties (density $\sigma$, transmissivity $\tau$, color emission $\kappa$).
*   **Output:** Final pixel color and opacity.

**Data Structures:**
*   **Vertex Data:** Stored in 3D textures [S2, S3, unofficial].
*   **Adjacency/Traversal Data:** Stored in 2D textures [S2, S3, unofficial].
*   **Lookup Table:** A precomputed table for blending, assuming a static mesh geometry [S4, unofficial].

**Algorithm Steps:**
The traversal is executed in a single render pass, concurrently for all pixels [S4, unofficial]. For each ray, the procedure follows these steps [S4, unofficial]:

1.  **Find Entry Point:** Determine the entry point $p_0$ of the ray into the convex hull of the mesh.
2.  **Find Exit Point and Calculate Density:** Determine the exit point $p_1$ of the current tetrahedron. Calculate the density values within the segment $[p_0, p_1]$.
3.  **Blend Fragment:** Blend the new fragment using the cell density and the distance between $p_0$ and $p_1$. This step typically uses a precomputed lookup table [S4, unofficial].
4.  **Update Position:** Set $p_1$ as the new $p_0$ for the next iteration.
5.  **Loop:** Return to Step 2.

**Mathematical Formulation:**
The contribution $E_i$ of the $i$-th segment (bounded by ray distances $s_i$ and $s_{i+1}$, with length $\Delta s_i = s_{i+1} - s_i$) to the final irradiance $E$ is calculated by integrating the volume rendering integral [S1, book]:

$$
E_i = \int_{s_i}^{s_{i+1}} \tau_i c_i T(s) ds
$$

Where:
*   $\tau_i$ is the transmissivity of the $i$-th tetrahedron.
*   $c_i$ is the color/emission of the $i$-th tetrahedron.
*   $T(s)$ is the transmission coefficient up to distance $s$, defined as $T(s) = \exp\left(-\int_0^s \tau(u) \sigma(u) du\right)$ [S1, book].
*   Within the tetrahedron, optical properties $\tau_i$ and $c_i$ are assumed constant [S1, book].

Substituting and separating the transparency up to $s_i$:

$$
E_i = \tau_i c_i \int_{s_i}^{s_{i+1}} \exp\left(-\int_0^{s_i} \tau(u) \sigma(u) du\right) \cdot \exp\left(-\int_{s_i}^s \tau(u) \sigma(u) du\right) ds
$$

Since $\tau_i$ and $\sigma_i$ are constant within the cell, this simplifies to:

$$
E_i = \tau_i c_i T(s_i) \int_{s_i}^{s_{i+1}} \exp\left(-\tau_i \sigma_i (s - s_i)\right) ds
$$

**Parameters:**
*   **Convexification:** The mesh must be convexified. Exterior cells are marked as "imaginary" and ignored during rendering [S2, S3, unofficial].
*   **Lookup Table:** Requires precomputation based on static mesh geometry [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Convex Meshes:** The algorithm strictly handles only convex meshes. It cannot find the re-entry of rays that leave the mesh temporarily [S2, S3, unofficial].
*   **Local Intersections:** If a tetrahedron is flipped and cuts into another tetrahedron's volume (local intersection), the algorithm fails to detect it because it jumps to neighbors without analyzing the neighborhood. This results in an incomplete fragment list and invalid pixel color [S4, unofficial].
*   **Dynamic Meshes:** The precomputed lookup table assumes a static mesh. For dynamic volumes (e.g., smoke), this is infeasible, leading to artifacts due to arbitrary distortion over time [S4, unofficial].
*   **Global Intersections:** The method does not handle global intersections (penetration of non-adjacent tetrahedra) correctly, leading to incomplete fragment lists [S4, unofficial].

## Subtopics

### Local Intersection Handling in Ray Traversal
Local intersection (or self-intersection) occurs when a tetrahedron is flipped and cuts into another tetrahedron's volume [S4, unofficial]. This is a bottleneck in ray tracing where the algorithm jumps from one cell to another without looking at other cells [S4, unofficial]. Because the algorithm is not aware of local intersections and does not analyze the neighborhood, the intersecting tetrahedron fails to contribute to the viewing ray, resulting in an invalid pixel color due to an incomplete list of fragments [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra intersected by the ray, executed in parallel for all pixels [S4, unofficial].
*   **Memory Complexity:** Requires storage for vertex data (3D textures) and adjacency data (2D textures) [S2, S3, unofficial]. Global random access memory is required for implementation [S4, unofficial].
*   **Performance:** GPU performance outpaces CPU, leading to good scalability [S2, S3, unofficial].
*   **Strengths:** Fast, concurrent traversal; no CPU-GPU data transfer; accurate interpolation of scalar data before transfer function application [S1, book][S4, unofficial].
*   **Limitations:** Strictly convex meshes only; cannot handle re-entry; precomputed lookup table limits dynamic use; fails on local and global intersections [S2, S3, S4, unofficial].
*   **When to Use:** Static, convex tetrahedral meshes where high performance is required and dynamic deformation is not expected [S4, unofficial].
*   **When Not to Use:** Dynamic volumes (e.g., smoke), non-convex meshes, or meshes with self-intersections [S2, S3, S4, unofficial].

## Background
The method is based on the work of Garrity [15], who showed how to trace a ray through a tetrahedral grid by determining the exit face and following neighbor pointers [S1, book]. Weiler et al. [56] (also cited as [12] and [66]) implemented this on a GPU [S1, book][S2, S3, S4, unofficial]. The convexification technique using imaginary cells was originally proposed by Williams [70] [S2, S3, unofficial]. Convexification of unstructured meshes remains a hard and currently unsolved problem with many unresolved issues [S2, S3, unofficial].

## Sources
[S1] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S3] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S4] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
