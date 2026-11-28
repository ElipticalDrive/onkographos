# Regular-Adaptive Octree Construction

Also called: Adaptive Construction, Adaptive Octree Construction, Adaptive vs Regular Octree Construction, Build Octree algorithm, Octree Construction with Complex Cell Criteria, Octree construction modes, Regular Construction, Top-bottom Octree Construction

## Overview
Regular-Adaptive Octree Construction is an iterative top-bottom algorithm that builds a Sparse Voxel Octree (SVO) which remains uniformly subdivided up to a specified minimal level (`minLevel`) and adapts its resolution based on surface topology and geometry between `minLevel` and a maximal level (`maxLevel`) [S2, unofficial][S3]. It solves the problem of generating manifold surfaces from implicit or voxelized data by ensuring that no leaf cell intersecting the surface boundary $\partial V$ is "complex," thereby preventing non-manifold configurations [S3]. It differs from purely dynamic growth algorithms (like its parent [Dynamic Octree Growth Algorithm](2375-dynamic-octree-growth-algorithm.md)) by enforcing a fixed regular structure at shallow depths and using explicit topological and curvature criteria for subdivision rather than simple occupancy checks [S2, unofficial][S3].

## How it works
**Inputs:**
*   Root cell $c$ of the octree [S2, unofficial][S3].
*   Minimal subdivision level `minLevel` (typically 2 or 3) [S2, unofficial][S3].
*   Maximal subdivision level `maxLevel` [S2, unofficial][S3].
*   Implicit surface function $F(x)$ or boundary $\partial V$ definition [S2, unofficial][S3].
*   Curvature threshold parameter $\delta \in [0, 1]$ [S3].

**Outputs:**
*   An octree that is regular (uniformly subdivided) up to level `minLevel` [S2, unofficial][S3].
*   An adaptive octree between `minLevel` and `maxLevel` where no leaf cell intersecting $\partial V$ is complex [S2, unofficial][S3].

**Data Structures:**
*   `toProcess`: A list or queue of cells to be evaluated [S2, unofficial][S3].
*   Cell structure: Contains level, spatial bounds, and status (leaf/branch) [G].

**Definitions:**
1.  **Complex Cell:** A cell $C$ is complex if at least one of its faces is complex [S3].
2.  **Complex Face:** A face is complex if: (1) It intersects $\partial V$ and all its vertices are either inside or outside $V$ [S3]. (2) At least one of its edges intersects $\partial V$ more than once [S3].
3.  **Inhomogeneous:** The cell contains parts of the surface (intersection points exist) [S3].
4.  **High Curvature:** Determined by Rule 2 below [S3].

**Algorithm Steps:**
1.  Initialize `toProcess` with the root cell $c$ [S2, unofficial][S3].
2.  While `toProcess` is not empty:
    a.  Retrieve `currentCell` from `toProcess` [S2, unofficial][S3].
    b.  Let $n$ be the level of `currentCell` [S3].
    c.  If $n < \text{minLevel}$: Subdivide unconditionally to maintain the regular structure [S2, unofficial][S3].
    d.  If $n > \text{maxLevel}$: Stop subdividing this branch and mark as leaf [S2, unofficial].
    e.  If $\text{minLevel} \le n \le \text{maxLevel}$:
        i.  Check if `currentCell` is **Complex**. If **Yes**, set `divide = true` [S3].
        ii. If **No**, check if `currentCell` is **Inhomogeneous** [S3].
        iii. If **Yes**, check if it has **High Curvature** using Rule 2 [S3]. If **Yes**, set `divide = true` [S3].
        iv. If **No** (not inhomogeneous or low curvature), mark `currentCell` as a **compact leaf** [S3].
    f.  If `divide` is true: Subdivide `currentCell` into 8 children $\{q_1, ..., q_8\}$ and add each child to `toProcess` [S2, unofficial][S3].

**Formulas:**
*   **Rule 1 (Complexity):** Subdivide if $C$ is complex [S3].
*   **Rule 2 (Curvature):** Let $I_i$ ($i=0...n$) be the intersection points of cell edges with $\partial V$ [S3]. Let $n_i$ be the normals at $I_i$. Subdivide if:
    $$ \min_{i \neq j} (n_i \cdot n_j) \le \delta $$
    where $\delta \in [0, 1]$ is the curvature threshold [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_regular_adaptive_octree(root_cell, min_level, max_level, delta):
    to_process = [root_cell]
    
    while to_process:
        current_cell = to_process.pop(0)
        n = current_cell.level
        
        # Regular subdivision phase
        if n < min_level:
            subdivide_and_enqueue(current_cell, to_process)
            continue
                
        # Max depth reached
        if n > max_level:
            current_cell.mark_as_leaf()
            continue
                
        # Adaptive subdivision phase
        divide = False
            
        # Rule 1: Check for complex cells
        if is_complex(current_cell):
            divide = True
        else:
            # Rule 2: Check for high curvature if inhomogeneous
            if is_inhomogeneous(current_cell):
                if has_high_curvature(current_cell, delta):
                    divide = True
                else:
                    current_cell.mark_as_compact_leaf()
            else:
                # Not inhomogeneous, safe to be a leaf
                current_cell.mark_as_compact_leaf()
                    
        if divide:
            subdivide_and_enqueue(current_cell, to_process)

def subdivide_and_enqueue(cell, queue):
    children = cell.divide_into_8_children()
    for child in children:
        queue.append(child)

def is_complex(cell):
    for face in cell.faces:
        if is_face_complex(face):
            return True
    return False

def is_face_complex(face):
    intersects_boundary = face.intersects_boundary()
    if intersects_boundary:
        vertices_inside = [v for v in face.vertices if evaluate_implicit(v) > 0]
        vertices_outside = [v for v in face.vertices if evaluate_implicit(v) < 0]
        if len(vertices_inside) == 4 or len(vertices_outside) == 4:
            return True
    for edge in face.edges:
        if edge.count_intersections_with_boundary() > 1:
            return True
    return False

def is_inhomogeneous(cell):
    return cell.has_intersection_with_boundary()  # [G] Checks if cell contains surface intersection points

def has_high_curvature(cell, delta):
    intersections = cell.get_edge_intersections_with_boundary()
    if len(intersections) < 2:
        return False
    normals = [compute_normal_at(p) for p in intersections]
    min_dot = float('inf')
    for i in range(len(normals)):
        for j in range(i + 1, len(normals)):
            dot = sum(a * b for a, b in zip(normals[i], normals[j]))
            if dot < min_dot:
                min_dot = dot
    return min_dot <= delta

def evaluate_implicit(point):
    return F(point)  # [G] Standard evaluation of implicit surface function

def compute_normal_at(point):
    return gradient_F(point)  # [G] Standard gradient computation for surface normal
```

**Edge Cases and Pitfalls:**
*   **Collapsed Cells:** Cells not subdivided to `maxLevel` are marked as "collapsed." These can cause non-manifold configurations if adjacent to higher-resolution cells [S2, unofficial].
*   **Precision:** The curvature check relies on accurate normal estimation at intersection points [S3].
*   **Boundary Conditions:** The algorithm assumes the root cell bounds the volume of interest appropriately [S2, unofficial][S3].

## Subtopics

### Local refinement algorithm
A recursive post-processing algorithm used to fix topological problems (non-manifold configurations) caused by collapsed cells adjacent to higher-resolution cells [S2, unofficial]. It identifies affected cells adjacent to collapsed cells and subdivides them until complex faces are eliminated [S2, unofficial]. It operates on a set of affected cells $A$ (initially containing collapsed cells $T$) and recursively expands $A$ by checking faces of cells in $A$ for complexity, adding adjacent cells to $A$ if complex faces are found [S2, unofficial]. The complexity is linear with respect to the number of affected cells [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The main construction is iterative [S2, unofficial]. The local refinement step has linear complexity over the number of cells affected by induced subdivision [S2, unofficial].
*   **Memory Complexity:** Depends on the depth of the octree and the surface complexity [S2, unofficial][S3]. The regular phase ensures a predictable base memory usage [S2, unofficial].
*   **Measured Performance:** Regular Construction (Serial) takes 129s at Level 6 and 226s at Level 7 [S1, unofficial]. Regular Construction (Parallel, 8 threads) takes ~23s at Level 6 and 78s at Level 7 [S1, unofficial].
*   **Strengths:** Guarantees manifold surfaces by eliminating complex cells [S3]. The regular phase captures general features efficiently [S3].
*   **Limitations:** Requires explicit topological checks (complexity, curvature) which are more expensive than simple occupancy checks [S2, unofficial][S3]. The local refinement step adds overhead if many collapsed cells exist [S2, unofficial].
*   **When to use:** When manifold surface extraction is required from implicit or voxel data, and topological correctness is critical [S2, unofficial][S3].
*   **When not to use:** When performance is prioritized over strict manifold guarantees, or when the surface is simple enough that standard occupancy-based SVO construction suffices [S1, unofficial].

## Background
The method originates from research on adaptive surface extraction from anisotropic volumetric data [S3]. It was developed to address the limitations of standard octree constructions that could produce non-manifold surfaces due to complex cell configurations [S2, unofficial][S3]. The approach combines a regular subdivision phase for efficiency with adaptive rules based on topological (complexity) and geometric (curvature) criteria [S2, unofficial][S3]. The local refinement algorithm was introduced to handle edge cases where collapsed cells interact with higher-resolution neighbors [S2, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
[S2] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2012/38588/, fetched 2026-10-28 (unofficial)
[S3] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
