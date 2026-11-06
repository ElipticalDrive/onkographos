# Hermite Dual Marching Cubes

Also called: Dual Marching Cubes with Hermite data, Hermite Dual Contouring, Hermite extension to Dual MC, Nielson's Dual Marching Cubes with QEF

## Overview
Hermite Dual Marching Cubes is a surface extraction method that combines the topological guarantees of Dual Marching Cubes with the feature-preserving capability of Dual Contouring [S1][S2, unofficial]. It solves the problem of non-manifold vertices and edges that arise in standard Dual Contouring when processing ambiguous sign configurations on uniform grids, while simultaneously reproducing sharp features that standard Dual Marching Cubes fails to capture [S1][S2, unofficial]. Unlike standard Dual Contouring, which may produce non-manifold topology, this method always produces a topological manifold [S1][S2, unofficial].

## How it works
**Inputs:**
- A uniform 3D grid of scalar field values $f(v)$ at vertices $v$.
- Hermite data associated with grid edges: for each edge $e$ intersecting the isosurface, the intersection point $p_e$ and the normal vector $n_e$ at that point.

**Outputs:**
- A manifold triangular or quadrilateral mesh representing the isosurface.

**Data Structures:**
- **Grid:** A 3D array of scalar values.
- **Edge Data:** For each grid edge, store the intersection point $p_e \in \mathbb{R}^3$ and normal $n_e \in \mathbb{R}^3$ if the edge crosses the isosurface (sign change).
- **Lookup Table:** A modified Marching Cubes lookup table [26] that maps each of the 256 cell configurations to a set of edge cycles. If a configuration is non-problematic, it maps to an empty set or a single trivial cycle. If problematic, it maps to specific cycles of edges [S2, unofficial].

**Steps:**

1. **Identify Sign-Change Edges:**
   For each cell in the grid, determine which edges intersect the isosurface by checking for sign changes in the scalar field values at the cell's vertices.

2. **Determine Contour Topology:**
   For each cell, use the modified Marching Cubes lookup table to identify the cycles of edges that form the contour components within the cell [S2, unofficial]. Each cycle consists of a list of edges on the cubic cell. Each edge is associated with exactly one vertex in the dual mesh [S2, unofficial].

3. **Construct Quadratic Error Functions (QEF):**
   For each identified cycle (which represents a contour component within the cell), construct a Quadratic Error Function (QEF) using the Hermite data (intersection points and normal vectors) from all edges associated with that cycle [S1][S2, unofficial].
   
   Let $C$ be a cycle of edges. For each edge $e \in C$, let $p_e$ be the intersection point and $n_e$ be the normal vector. The QEF for the vertex $v_C$ associated with cycle $C$ is defined as:
   $$ E(v) = \sum_{e \in C} ((v - p_e) \cdot n_e)^2 $$
   This minimizes the distance from $v$ to the planes defined by the Hermite data [S1][S2, unofficial].

4. **Minimize QEF to Place Vertices:**
   Place the vertex $v_C$ for each cycle $C$ at the location that minimizes the constructed QEF [S1][S2, unofficial]. This is a least-squares problem. The optimal vertex position $v^*$ is the solution to:
   $$ v^* = \arg \min_v E(v) $$
   This can be solved by setting the gradient of $E(v)$ to zero:
   $$ \nabla E(v) = 2 \sum_{e \in C} ((v - p_e) \cdot n_e) n_e = 0 $$
   $$ \left( \sum_{e \in C} n_e n_e^T \right) v = \sum_{e \in C} (p_e \cdot n_e) n_e $$
   Let $A = \sum_{e \in C} n_e n_e^T$ and $b = \sum_{e \in C} (p_e \cdot n_e) n_e$. Then $v^* = A^{-1} b$ [G]. If $A$ is singular (e.g., all normals are parallel), a pseudo-inverse or regularization may be needed [G].

5. **Generate Polygons:**
   Generate polygons by connecting the vertices associated with each sign-change edge across the four adjacent cells sharing that edge [S2, unofficial]. This creates a quadrilateral surface dual to the Marching Cubes surface [S2, unofficial]. Each edge in the primal grid corresponds to one quadrilateral in the dual mesh, formed by the vertices placed in the four cells sharing that edge.

**Parameters:**
- Isosurface value: Typically 0 for signed distance fields.
- QEF Solver: Direct linear solver (e.g., Cholesky decomposition) for small systems (3x3 matrix).

**Edge Cases and Pitfalls:**
- **Singular QEF:** If all normals in a cycle are parallel, the matrix $A$ is singular. This may occur in flat regions or degenerate configurations. A small regularization term $\epsilon I$ can be added to $A$ [G].
- **Boundary Conditions:** Cells on the boundary of the grid may have fewer than four adjacent cells. The polygon generation step must handle these cases by only connecting vertices in existing adjacent cells [G].
- **Precision:** The QEF minimization involves solving a linear system. Numerical stability should be considered, especially if normals are nearly parallel.

## Subtopics

### Dual Marching Cubes with Adaptive Octree and QEF Feature Isolation
- **What it is:** An extension that uses an adaptive octree to vary resolution and isolates features using QEF.
- **Difference:** Uses adaptive resolution instead of uniform grid; focuses on feature isolation.
- **When to choose:** For models with varying detail levels.
- **Link:** [Dual Marching Cubes with Adaptive Octree and QEF Feature Isolation](2291-dual-marching-cubes-with-adaptive-octree-and-qef-feature-iso.md)

### Dual Marching Cubes (Counting Pass)
- **What it is:** A variant that uses a counting pass for parallel implementation.
- **Difference:** Optimized for GPU parallelism via counting pass.
- **When to choose:** For high-performance GPU implementations.
- **Link:** [Dual Marching Cubes (Counting Pass)](3766-dual-marching-cubes-counting-pass.md)

### Dual Isosurfacing with Analytical Trilinear Intersection
- **What it is:** Uses analytical trilinear intersection for more accurate edge intersections.
- **Difference:** More accurate intersection points compared to linear interpolation.
- **When to choose:** When high accuracy is critical.
- **Link:** [Dual Isosurfacing with Analytical Trilinear Intersection](3798-dual-isosurfacing-with-analytical-trilinear-intersection.md)

### Curvature/Normal Similarity Loss
- **What it is:** A loss function for training neural fields to preserve curvature and normals.
- **Difference:** Used in neural rendering, not traditional grid-based methods.
- **When to choose:** For neural implicit surfaces.
- **Link:** [Curvature/Normal Similarity Loss](4354-curvature-normal-similarity-loss.md)

### Geometric Self-Intersection Elimination
- **What it is:** Post-processing step to remove self-intersections.
- **Difference:** Not a contouring method, but a cleanup step.
- **When to choose:** When self-intersections are present.
- **Link:** [Geometric Self-Intersection Elimination](4757-geometric-self-intersection-elimination.md)

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of grid cells, as each cell is processed independently. The QEF minimization is a constant-time operation (solving a 3x3 system) per cycle.
- **Memory Complexity:** $O(N)$ for storing grid data and edge information. The output mesh size is proportional to the number of edges.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with grid resolution. Parallelization is straightforward as cells are independent.
- **Strengths:** Always produces a manifold surface; reproduces sharp features; simple implementation on uniform grids.
- **Limitations:** Uniform grid may be inefficient for flat regions; QEF minimization may be numerically unstable in degenerate cases.
- **When to use:** When manifold topology and sharp features are both required on uniform grids.
- **When not:** When adaptive resolution is needed or when high accuracy in intersection points is critical (consider analytical intersection methods).

## Background
Hermite Dual Marching Cubes was introduced by Nielson et al. in 2007 as an extension of Dual Marching Cubes to incorporate Hermite data [S1][S2, unofficial]. It addresses the limitations of standard Dual Contouring, which can produce non-manifold topology on uniform grids, and standard Marching Cubes, which fails to reproduce sharp features [S1][S2, unofficial]. The method builds on the topology guarantees of Dual Marching Cubes and the feature-preserving capability of Dual Contouring [S1][S2, unofficial].

## Sources
[S1] Manifold Dual Contouring, https://doi.org/10.1109/tvcg.2007.1012, fetched 2026-10-26
[S2] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
