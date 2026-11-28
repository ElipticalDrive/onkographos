# Garrity Ray Casting
Also called: Garrity, Garrity's Ray Casting
## Overview
Garrity Ray Casting is a method for rendering unstructured meshes by exploiting intrinsic mesh connectivity to optimize ray traversal [S1, unofficial][S2, unofficial]. It solves the problem of efficiently traversing non-convex domains where rays may exit and re-enter the mesh multiple times [S1, unofficial][S2, unofficial].
## How it works
**Inputs:** An unstructured mesh defined by cells and their connectivity; a set of rays defined by origin and direction [G].
**Outputs:** A rendered image where each pixel color is accumulated based on the ray's intersection with the mesh cells [G].
**Data Structures:**
1. Mesh Connectivity: Explicit adjacency lists or pointers linking each cell to its neighboring cells [G]. Exact layouts, fields, sizes, and bit packing are not given in the sources [G].
2. Hierarchical Spatial Data Structure: A structure storing only the boundary cells of the mesh to accelerate re-entry queries [S1, unofficial][S2, unofficial]. Exact layouts, fields, sizes, and bit packing are not given in the sources [G].
**Steps:**
1. Initialization: Determine the initial entry point of the ray into the mesh [G].
2. Cell Traversal: Identify the current cell containing the ray and use the mesh connectivity to determine the exit face along the ray's direction [S1, unofficial][S2, unofficial].
3. Intersection Calculation: Compute the intersection point $P_{exit}$ between the ray and the exit face [G].
4. Volume Integration: Accumulate optical properties along the segment $[P_{entry}, P_{exit}]$ using the cell's volume data [G].
5. Neighbor Selection: Identify the next cell. In convex regions, this is the neighbor sharing the exit face [G].
6. Re-entry Handling: If the ray exits the mesh domain, query the hierarchical spatial data structure to find the next intersecting boundary cell, avoiding expensive global searches [S1, unofficial][S2, unofficial].
7. Iteration: Update $P_{entry} = P_{exit}$ and repeat from Step 2 until the ray permanently exits the mesh or reaches a maximum depth [G].
**Formulas:** Not given in the sources. Standard ray-face intersection is used: $t = \frac{(P_{face} - O) \cdot N}{D \cdot N}$, where $O$ is the ray origin, $D$ is the direction, $N$ is the face normal, and $P_{face}$ is a point on the face [G].
**Parameters:** Not given in the sources [G].
**Edge Cases and Pitfalls:**
* Non-Convexity: Rays may exit and re-enter the mesh multiple times, requiring the spatial structure for efficient re-entry [S1, unofficial][S2, unofficial].
* Degenerate Cells: Zero-volume cells or coplanar faces may cause numerical instability in intersection calculations [G].
* Boundary Precision: Floating-point errors at cell boundaries may cause rays to miss neighbors or re-enter the same cell [G]. A small epsilon offset is typically applied to the exit point to ensure it lies within the next cell [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def garrity_ray_casting(mesh, rays):
    boundary_tree = build_boundary_tree(mesh) # [G] Hierarchical spatial structure for boundary cells
    for ray in rays:
        entry = find_initial_entry(mesh, ray) # [G] Initial entry point
        current = mesh.cell_containing(entry) # [G] Locate starting cell
        while current is not None:
            exit_face = find_exit_face(current, ray) # [G] Use mesh connectivity
            t = ray_intersect_face(ray, exit_face) # [G] Standard ray-face intersection
            exit_point = ray.origin + t * ray.direction
            accumulate_volume_data(current, entry, exit_point) # [G] Integrate optical properties
            next_cell = current.neighbors[exit_face] # [G] Convex neighbor lookup
            if next_cell is None: # [G] Ray exited mesh domain
                current = query_boundary_tree(boundary_tree, ray, exit_point) # [G] Find next boundary cell
                if current is None:
                    break
            entry = exit_point
```
## Subtopics

### [Irregular Data Ray Traversal](10329-irregular-data-ray-traversal.md)
Renders curvilinear or unstructured data by determining an initial cell and iteratively traversing subsequent cells along a ray. It differs from regular grid traversal by relying on explicit mesh connectivity and acceleration structures over boundary faces rather than implicit coordinate mapping. Choose this when dealing with complex, non-hierarchical unstructured grids where connectivity is the primary traversal aid [G].

## Operating characteristics
* Time Complexity: Not given in the sources [G].
* Memory Complexity: Not given in the sources [G].
* Strengths: Efficient for meshes with high connectivity; avoids global searches for re-entry [S1, unofficial][S2, unofficial].
* Limitations: Requires explicit connectivity; spatial structure construction overhead; less effective for highly fragmented or disconnected meshes without careful boundary definition [G].
* When to Use: Rendering unstructured grids where connectivity is known and rays traverse many cells [G].
* When Not to Use: When the mesh is highly non-convex with frequent re-entries and the spatial structure is too large to fit in memory, or when connectivity is not available [G].
## Background
Garrity [28] pioneered the use of ray casting for rendering unstructured meshes [S1, unofficial][S2, unofficial]. The technique was developed to address the inefficiencies of naive ray casting on irregular grids by leveraging the mesh's intrinsic connectivity [S1, unofficial][S2, unofficial]. It is cited in surveys of GPU-based volume rendering of unstructured grids as a foundational method for connectivity-based traversal [S1, unofficial][S2, unofficial].
## Sources
[S1] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey of GPU-Based Volume Rendering of Unstructured Grids, fetched 2026-10-28 (unofficial)
