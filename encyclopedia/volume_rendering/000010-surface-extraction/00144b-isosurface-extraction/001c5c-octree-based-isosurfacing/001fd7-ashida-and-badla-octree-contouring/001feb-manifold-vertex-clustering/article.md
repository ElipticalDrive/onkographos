# Manifold Vertex Clustering

Also called: Manifold Vertex Clustering Criterion, Topology criterion for collapsible vertices, Topology-Constrained Clustering

## Overview
Manifold Vertex Clustering is a topological constraint applied to vertex clustering in adaptive isosurface extraction to ensure the resulting mesh is a closed 2-manifold [S1][S2, unofficial]. It solves the problem of non-manifold artifacts (such as edges shared by more than two polygons or vertices with non-disk-like neighborhoods) that arise when collapsing multiple fine-resolution vertices into a single coarse vertex [S1][S2, unofficial]. It sets itself apart from standard vertex clustering by adding a sufficient topological check—based on the Euler characteristic and edge intersection counts—rather than relying solely on geometric error metrics like the Quadratic Error Function (QEF) [S1][S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A vertex $v$ on the simplified contour.
- $C_v$: The octree cell in which $v$ was created by clustering [S1][S2, unofficial].
- $S_v$: The set of all polygons on the unsimplified surface (extracted at the finest level of the octree) incident on vertices that are clustered to $v$ [S1][S2, unofficial].

**Outputs:**
- A boolean indicating whether $v$ is valid for clustering (i.e., satisfies the manifold criterion) [S1][S2, unofficial].
- Precomputed values for each vertex $v$: $\chi(S_v)$ and $e_i(S_v)$ for $i=1,\dots,12$ [S1][S2, unofficial].

### Data Structures
For each vertex $v$, store 13 numbers:
1. $\chi(S_v)$: The Euler characteristic of the surface portion $S_v$ [S1][S2, unofficial].
2. $e_i(S_v)$: The number of intersections of $S_v$ with the $i$-th edge of the octree cell $C_v$, for $i=1,\dots,12$ [S1][S2, unofficial].

### Numbered Steps

1. **Base Case (Leaf Cell):**
   For a vertex $v$ in a leaf cell:
   - $S_v$ consists of a single vertex $v$ connected to polygons dual to the edges associated with $v$ [S1][S2, unofficial].
   - Set $e_i(S_v) = 1$ for each edge associated with $v$, and $0$ otherwise [S1][S2, unofficial].
   - Set $\chi(S_v) = 1$ [S1][S2, unofficial].

2. **Recursive Intersection Count:**
   For a clustered vertex $v$ formed by clustering child vertices $v_k$ from the child cells of $C_v$:
   - Compute $e_i(S_v)$ by summing the intersections of each child surface $S_{v_k}$ along the edges of $C_v$ [S1][S2, unofficial].
   - Specifically, for each edge $i$ of $C_v$:
     $$ e_i(S_v) = \sum_k e_i(S_{v_k}) $$
     where $e_i(S_{v_k})$ is the precomputed intersection count of $S_{v_k}$ on edge $i$ [S1][S2, unofficial].

3. **Recursive Euler Characteristic:**
   Compute $\chi(S_v)$ using the inductive formula [S1][S2, unofficial]:
   $$ \chi(S_v) = \sum_k \chi(S_{v_k}) - \frac{\sum_k e(S_{v_k})}{4} $$
   where:
   - $v_k$ are the vertices clustered to form $v$ [S1][S2, unofficial].
   - $e(S_{v_k})$ is the sum of intersections of $S_{v_k}$ along the **internal** edges of $C_v$ [S1][S2, unofficial].
   - The internal edges of $C_v$ are the 4 edges shared between child cells within $C_v$ [S1][S2, unofficial].

4. **Manifold Criterion Check:**
   A vertex $v$ is valid for clustering (generates a closed 2-manifold) if and only if:
   1. $\chi(S_v) = 1$ [S1][S2, unofficial].
   2. For each of the 6 faces of $C_v$, the number of intersections of $S_v$ with the 4 edges of that face is either 0 or 2 [S1][S2, unofficial].
      - Note: The criterion applies to the intersections on the edges of the faces. Since each edge belongs to two faces, the condition is checked per face's boundary edges [S1][S2, unofficial].

### Formulas
- **Euler Characteristic Definition:**
  $$ \chi(S_v) = V(S_v) - E(S_v) + F(S_v) $$
  where $V(S_v)$, $E(S_v)$, and $F(S_v)$ are the numbers of vertices, edges, and faces of $S_v$ [S1][S2, unofficial].

- **Recursive Euler Characteristic Update:**
  $$ \chi(S_v) = \sum_k \chi(S_{v_k}) - \frac{\sum_k e(S_{v_k})}{4} $$
  where $e(S_{v_k}) = \sum_{j \in \text{internal edges of } C_v} e_j(S_{v_k})$ [S1][S2, unofficial].

### Parameters
- $C_v$: The octree cell containing vertex $v$ [S2, unofficial].
- $S_v$: The set of polygons from the finest level incident on vertices clustered to $v$ [S2, unofficial].

### Edge Cases and Pitfalls
- **$\chi(S_v) \neq 1$:** Clustering may remove a surface component or result in a non-manifold vertex [S1][S2, unofficial]. For example, if $S_v$ is a sphere ($\chi=2$) or cylinder ($\chi=0$), clustering is invalid [S1][S2, unofficial].
- **Violation of Intersection Condition:** If the number of intersections of $S_v$ with the edges of a face of $C_v$ is not 0 or 2 (e.g., 4 intersections), clustering may introduce a non-manifold edge shared by four polygons [S1][S2, unofficial].
- **Sufficiency:** The two conditions are sufficient but not necessary [S1][S2, unofficial].
- **Genus Preservation:** Since $\chi(S_v)=1$ implies the genus of $S_v$ is zero, vertex clustering preserves the genus of the unsimplified surface [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_manifold_properties(v, octree_cell):
    """
    Computes chi(S_v) and e_i(S_v) for vertex v in octree_cell C_v.
    Assumes a bottom-up traversal from leaf cells to root.
    """
    if octree_cell.is_leaf():
        # Base case: S_v is a single vertex connected to polygons dual to edges
        chi = 1
        e = [0] * 12  # 12 edges in an octree cell
        # Set e_i = 1 for edges associated with v, 0 otherwise
        for edge_idx in v.associated_edges:
            e[edge_idx] = 1
        return chi, e

    else:
        # Recursive case: v is formed by clustering child vertices v_k
        chi_sum = 0
        e_total = [0] * 12
        e_internal_sum = 0  # Sum of intersections on internal edges of C_v
        
        # Identify child vertices v_k that were clustered to form v
        child_vertices = get_clustered_children(v)
        
        for v_k in child_vertices:
            child_cell = get_child_cell_containing(v_k, octree_cell)
            chi_k, e_k = compute_manifold_properties(v_k, child_cell)
            
            chi_sum += chi_k
            
            # Sum intersections for all 12 edges
            for i in range(12):
                e_total[i] += e_k[i]
                
            # Sum intersections on internal edges of C_v
            # Internal edges are those shared between child cells within C_v
            # Assuming edges 0-3 are internal (example indexing, adjust to actual octree edge layout)
            # The source specifies "internal edges of C_v". In an octree, 
            # internal edges are the 4 edges meeting at the center of the cell faces? 
            # Actually, the source says "internal edges of C_v" in the context of 
            # summing e(S_vk). Let's assume we have a list of internal edge indices.
            for internal_edge_idx in get_internal_edges(octree_cell):
                e_internal_sum += e_k[internal_edge_idx]
                
        # Compute chi(S_v) using the inductive formula
        chi_v = chi_sum - (e_internal_sum / 4)
        
        return chi_v, e_total

def check_manifold_criterion(v, octree_cell, chi_v, e_v):
    """
    Checks if vertex v satisfies the manifold criterion.
    """
    # Condition 1: Euler characteristic must be 1
    if chi_v != 1:
        return False
        
    # Condition 2: For each of the 6 faces of C_v, 
    # the number of intersections of S_v with the 4 edges of that face is 0 or 2.
    faces = get_faces(octree_cell)
    for face in faces:
        edge_indices = get_face_edges(face)
        intersection_count = sum(e_v[i] for i in edge_indices)
        if intersection_count not in (0, 2):
            return False
            
    return True
```

## Subtopics

### [Adaptive Contouring](2117-ashida-and-badla-octree-contouring.md)
The base method that extracts contours directly from octrees. Manifold Vertex Clustering is a constraint applied on top of this method to ensure manifoldness.

### [Vertex Clustering](52-name.md)
The general technique of collapsing multiple vertices into one. Manifold Vertex Clustering adds topological checks to this general technique.

## Operating characteristics
- **Time Complexity:** The computation of $\chi(S_v)$ and $e_i(S_v)$ is done recursively during the bottom-up octree collapse. Each vertex requires constant time operations (summing 13 values) relative to its children, making the overhead linear in the number of vertices [S1][S2, unofficial].
- **Memory Complexity:** Each vertex stores 13 integers (or floats), which is a small constant overhead [S1][S2, unofficial].
- **Strengths:** Ensures the output mesh is a closed 2-manifold, preserving genus and avoiding non-manifold edges [S1][S2, unofficial]. Efficiently verifiable during clustering [S1][S2, unofficial].
- **Limitations:** The conditions are sufficient but not necessary, potentially rejecting valid collapses [S1][S2, unofficial]. Requires tracking the finest-level surface polygons for each clustered vertex, which may increase memory usage if not managed carefully [S1][S2, unofficial].
- **When to Use:** When a manifold mesh is required, such as for simulation or rendering, and adaptive resolution is used [S1][S2, unofficial].
- **When Not to Use:** When non-manifold geometry is acceptable or when performance is critical and the topological overhead is prohibitive [S1][S2, unofficial].

## Background
Manifold Vertex Clustering was introduced by Ashida and Badla in their 2007 paper "Manifold Dual Contouring" [S1][S2, unofficial]. It was developed to address the non-manifold artifacts that can occur in adaptive dual contouring when vertices are clustered [S1][S2, unofficial]. The method builds on the recursive nature of octree-based contouring to efficiently compute topological invariants [S1][S2, unofficial].

## Sources
[S1] Manifold Dual Contouring, https://doi.org/10.1109/tvcg.2007.1012, fetched 2026-10-26
[S2] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
