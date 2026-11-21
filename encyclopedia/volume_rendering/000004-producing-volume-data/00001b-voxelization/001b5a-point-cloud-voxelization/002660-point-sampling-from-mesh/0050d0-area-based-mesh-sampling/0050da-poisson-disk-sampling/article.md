# Poisson Disk Sampling

Also called: Blue Noise Point Sampling

## Overview
Poisson disk sampling is a surface sampling method that uniformly distributes points on a mesh's surface with a minimum geodesic distance constraint of $2r$ between them, given a disk of radius $r$ [S1]. It utilizes blue noise properties to reduce the effects of aliasing in the later steps of the resulting model [S1]. It differs from its parent, Area-Based Mesh Sampling, by enforcing a strict minimum distance constraint rather than relying solely on area-proportional probability, and from Fast Poisson Disk Sampling (2D) by operating on the curved surface of a 3D mesh rather than a flat 2D domain [S1, 16881-fast-poisson-disk-sampling-2d.md].

## How it works
**Inputs:** A triangular mesh $M$; a sampling radius $r$ defining the exclusion zone.
**Outputs:** A set of points $P$ on the mesh surface.

**Data Structures:**
1.  **Point Set $P$**: List of sampled points, each storing 3D coordinates $(x, y, z)$ and the triangle index $t$ where the point resides.
2.  **Active List $A$**: Queue of points yet to be processed for neighborhood expansion.
3.  **Spatial Hash Grid**: A grid with cell size $r$ (or slightly larger, e.g., $r/\sqrt{d}$ in $d$ dimensions, though for surface sampling, geodesic distance is key) to efficiently query neighbors. [G] Since geodesic distance is required, the grid serves as a coarse rejection filter for Euclidean distance before expensive geodesic computation.

**Steps:**
1.  **Initialization**: Create an empty point set $P$ and active list $A$.
2.  **Seed Selection**: Select a random point $p_0$ on the mesh surface. Add $p_0$ to $P$ and $A$.
3.  **Sampling Loop**: While $A$ is not empty:
    a. Remove a point $p$ from $A$.
    b. Generate $k$ candidate points $c_i$ uniformly distributed within the spherical shell of inner radius $r$ and outer radius $2r$ centered at $p$ [G]. Note: The constraint is minimum distance $2r$, so candidates must be at least $2r$ away from existing points. The shell is typically defined between $r$ and $2r$ relative to the *previous* point in some implementations, but strictly, to maintain the $2r$ minimum distance from *all* points, candidates are generated in the annulus $[r, 2r]$ from $p$ and then validated against all neighbors. [S1] specifies minimum distance $2r$.
    c. For each candidate $c_i$:
        i. Check if $c_i$ lies on the mesh surface. If not, discard. [G]
        ii. Check if $c_i$ is at least $2r$ away (geodesic distance) from all points in $P$. Use the spatial hash grid to find candidate neighbors within Euclidean distance $2r$. For each neighbor, compute the geodesic distance. If any neighbor is closer than $2r$, discard $c_i$.
        iii. If valid, add $c_i$ to $P$, add $c_i$ to $A$, and update the spatial hash grid.
4.  **Termination**: When $A$ is empty, return $P$.

**Parameters:**
-   $r$: Sampling radius. Typical values depend on mesh scale.
-   $k$: Number of candidates per active point. Typical value: 30 [G].

**Edge Cases:**
-   **Precision**: Geodesic distance computation can be sensitive to mesh quality. Degenerate triangles may cause failures.
-   **Boundaries**: On open meshes, points near boundaries may have fewer valid sampling locations.
-   **Empty Mesh**: If the mesh has no area, return empty set.

## Subtopics

## Operating characteristics
-   **Time Complexity**: $O(N \log N)$ or $O(N)$ depending on spatial hashing efficiency, where $N$ is the number of sampled points. Geodesic distance computation is expensive, often requiring precomputed shortest path trees or approximations.
-   **Memory Complexity**: $O(N)$ for storing points and grid.
-   **Strengths**: Produces blue noise distribution, reducing aliasing artifacts in rendering [S1]. Uniform coverage.
-   **Limitations**: Geodesic distance computation is slow. May fail to fill small regions if $r$ is too large relative to local geometry.
-   **When to use**: When blue noise properties are critical for reducing aliasing in volume rendering or texture mapping.
-   **When not to use**: When performance is critical and blue noise is not required; use Area-Based Mesh Sampling instead.

## Background
Poisson disk sampling was introduced as a method for generating blue noise point distributions [S1]. It is widely used in computer graphics for texture synthesis, point cloud generation, and anti-aliasing. The method ensures that points are well-spaced while maintaining a random appearance, which is desirable for avoiding regular grid artifacts.

## Sources
[S1] A Comparative Assessment Of Implicit and Explicit Plenoptic Scene Representations, https://inria.hal.science/hal-04709909v1, fetched 2026-10-26
