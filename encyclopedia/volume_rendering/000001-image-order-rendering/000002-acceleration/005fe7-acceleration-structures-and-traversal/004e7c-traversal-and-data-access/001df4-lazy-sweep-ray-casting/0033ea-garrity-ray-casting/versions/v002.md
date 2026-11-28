# Garrity Ray Casting

Also called: Garrity, Garrity's Ray Casting

## Overview
Garrity Ray Casting is a method for rendering unstructured meshes (such as tetrahedral or hexahedral grids) by exploiting intrinsic mesh connectivity to optimize ray traversal [S1, unofficial][S2, unofficial]. It solves the problem of efficiently traversing non-convex domains where rays may exit and re-enter the mesh multiple times, distinguishing itself from sibling methods like [Lazy Sweep Ray Casting](1649-lazy-sweep-ray-casting.md) by relying on explicit boundary cell indexing rather than sweep-plane paradigms.

## How it works
**Inputs:** An unstructured mesh defined by cells and their connectivity; a set of rays defined by origin and direction.
**Outputs:** A rendered image where each pixel color is accumulated based on the ray's intersection with the mesh cells.

**Data Structures:**
1.  **Mesh Connectivity:** Explicit adjacency lists or pointers linking each cell to its neighboring cells.
2.  **Hierarchical Spatial Data Structure:** A structure (e.g., a bounding volume hierarchy or grid) storing only the boundary cells of the mesh. This structure maps spatial regions to the set of boundary cells that intersect them [S1, unofficial][S2, unofficial].

**Steps:**
1.  **Initialization:** For each ray, determine the initial entry point into the mesh. This typically involves a coarse search or assuming the ray starts inside the domain. [G] If the starting position is unknown, a bounding box test against the entire mesh is required to find the first intersected cell.
2.  **Cell Traversal:** Identify the current cell containing the ray. Use the mesh connectivity to determine the exit face of the current cell along the ray's direction.
3.  **Intersection Calculation:** Compute the intersection point $P_{exit}$ between the ray and the exit face. The ray segment within the current cell is $[P_{entry}, P_{exit}]$.
4.  **Volume Integration:** Accumulate optical properties (color, opacity) along the segment $[P_{entry}, P_{exit}]$ using the cell's volume data.
5.  **Neighbor Selection:** Identify the next cell. In a convex mesh, this is the neighbor sharing the exit face. In a non-convex mesh, the ray may exit the entire mesh domain.
6.  **Re-entry Handling:** If the ray exits the mesh domain, use the **Hierarchical Spatial Data Structure** to find the next boundary cell that the ray intersects. This avoids expensive global searches [S1, unofficial][S2, unofficial].
7.  **Iteration:** Set $P_{entry} = P_{exit}$ and repeat from Step 2 until the ray exits the mesh permanently or reaches a maximum depth.

**Formulas:**
Let $R(t) = O + t \cdot D$ be the ray, where $O$ is the origin and $D$ is the direction.
Let $C_i$ be the current cell.
Let $F_{exit}$ be the face of $C_i$ intersected by $R(t)$ at the maximum $t > 0$.
The exit parameter $t_{exit}$ is found by solving the plane equation for $F_{exit}$:
$$ t_{exit} = \frac{(P_{face} - O) \cdot N}{D \cdot N} $$
where $N$ is the normal of $F_{exit}$ and $P_{face}$ is a point on the face.

**Parameters:**
*   **Spatial Structure Type:** Not specified in sources [S1, unofficial][S2, unofficial].
*   **Traversal Step Size:** Determined by cell geometry, not fixed.

**Edge Cases and Pitfalls:**
*   **Non-Convexity:** Rays may exit and re-enter the mesh multiple times. Failure to use the spatial structure for re-entry leads to $O(N)$ search costs per exit [S1, unofficial][S2, unofficial].
*   **Degenerate Cells:** Zero-volume cells or coplanar faces may cause numerical instability in intersection calculations.
*   **Boundary Precision:** Floating-point errors at cell boundaries may cause rays to miss neighbors or re-enter the same cell. [G] A small epsilon offset is typically applied to the exit point to ensure it lies within the next cell.

## Subtopics

### [Irregular Data Ray Traversal](10329-irregular-data-ray-traversal.md)
A generalization of Garrity's approach for arbitrary unstructured meshes, differing by its reliance on explicit mesh connectivity and boundary face acceleration structures rather than implicit coordinate mapping. Choose this when dealing with complex, non-hierarchical unstructured grids where connectivity is the primary traversal aid.

## Operating characteristics
*   **Time Complexity:** Traversal within a cell is $O(1)$. Re-entry cost depends on the spatial structure; typically $O(\log N)$ or $O(1)$ depending on the hierarchy. Total time is proportional to the number of cells intersected by rays.
*   **Memory Complexity:** $O(N)$ for mesh storage plus $O(B)$ for the boundary spatial structure, where $B$ is the number of boundary cells.
*   **Strengths:** Efficient for meshes with high connectivity; avoids global searches for re-entry.
*   **Limitations:** Requires explicit connectivity; spatial structure construction overhead; less effective for highly fragmented or disconnected meshes without careful boundary definition.
*   **When to Use:** Rendering unstructured grids where connectivity is known and rays traverse many cells.
*   **When Not to Use:** When the mesh is highly non-convex with frequent re-entries and the spatial structure is too large to fit in memory, or when connectivity is not available.

## Background
Garrity [28] pioneered the use of ray casting for rendering unstructured meshes [S1, unofficial][S2, unofficial]. The technique was developed to address the inefficiencies of naive ray casting on irregular grids by leveraging the mesh's intrinsic connectivity. It is cited in surveys of GPU-based volume rendering of unstructured grids as a foundational method for connectivity-based traversal [S1, unofficial][S2, unofficial].

## Sources
[S1] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
