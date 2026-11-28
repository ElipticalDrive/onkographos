# Hoppe's Progressive Mesh Simplification

Also called: Edge Collapse, Hoppe's Simplification, Mesh Simplification for Multiresolution, Progressive Mesh Simplification, Progressive Meshes

## Overview
Hoppe's Progressive Mesh Simplification is an iterative edge-collapse algorithm that generates a hierarchy of coarse meshes from a high-resolution input by preserving topological consistency while removing geometric details [S2]. It differs from its siblings by strictly enforcing manifold topology via Hoppe's consistency checks and using a simplified error metric based solely on edge length, prioritizing an even vertex distribution and low computational cost over the preservation of sharp features or optimized vertex positions [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh $M_0$ with $V$ vertices and $F$ faces [S2].
*   **Output:** A sequence of meshes $M_0, M_1, \dots, M_k$ where each $M_{i+1}$ is a simplification of $M_i$, and a data structure allowing reconstruction of the original mesh from any level $M_i$ [S2][S4, unofficial].

**Data Structures:**
1.  **Mesh Representation:** Adjacency lists storing vertex positions, face indices, and 1-ring neighbor connectivity [G].
2.  **Edge Heap:** A priority queue (ordered heap) storing all edges in the current mesh, keyed by their collapse error cost [S2].
3.  **Edge Metadata:** For each edge, store the computed error cost and a validity flag [S2].

**Error Metric:**
The error cost $E(e)$ for an edge $e$ is defined solely by its Euclidean length $L(e)$ [S2]:
$$ E(e) = L(e) $$
Unlike metrics that preserve sharp edges or curvature, this metric aims to remove details evenly across high-curvature regions to create a uniform vertex distribution [S2].

**Algorithm Steps:**
1.  **Initialization:** Compute the length $L(e)$ for every edge $e$ in $M_0$ [S2]. Insert all edges into the priority queue $H$ with key $E(e) = L(e)$ [S2].
2.  **Iteration:** While $H$ is not empty:
    a.  Pop the edge $e_{min}$ with the minimum error cost from $H$ [S2].
    b.  **Consistency Check:** Verify if collapsing $e_{min}$ maintains topological consistency using Hoppe's consistency checks (based on topological analysis of the 1-ring neighborhood) [S2].
    c.  **Isotropy Constraint:** Only collapse edges that are smaller than twice the maximum edge length in convex offset regions, $l_{max} = 2\delta \sin(\sigma_{max})$, to maintain isotropy [S1].
    d.  **Collapse:** If valid:
        i.  Collapse $e_{min}$ to its geometric midpoint $v_{mid} = \frac{v_1 + v_2}{2}$, where $v_1, v_2$ are the endpoints of $e_{min}$ [S2].
        ii. Update the 1-ring neighbor structure of the affected vertices to reflect the new connectivity (faces removed, new faces created) [S2].
        iii. Recompute the error costs for all edges incident to the modified 1-ring neighborhood [S2].
        iv. Update the positions of these edges in $H$ [S2].
    e.  If invalid, skip the edge and continue to the next iteration [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import heapq

def hoppe_simplification(mesh):
    # mesh: object with vertices, faces, and adjacency lists
    # Returns: list of meshes representing the hierarchy
    
    hierarchy = [mesh]
    current_mesh = mesh
    
    # Step 1: Initialize heap with edge lengths [S2]
    edge_heap = []
    for edge in current_mesh.edges:
        cost = edge.length()
        heapq.heappush(edge_heap, (cost, edge.id))
        
    while edge_heap:
        # Step 2: Pop edge with minimum cost [S2]
        cost, edge_id = heapq.heappop(edge_heap)
        edge = current_mesh.get_edge(edge_id)
        
        # Skip if edge was already collapsed or invalidated [G]
        if not edge.is_valid():
            continue
            
        # Step 3: Check topological consistency [S2]
        if not hoppe_consistency_check(current_mesh, edge):
            continue
            
        # Step 4: Isotropy constraint [S1]
        if edge.length() >= 2 * current_mesh.max_edge_length * 0.5: # Simplified check for lmax
            continue
            
        # Step 5: Collapse to midpoint [S2]
        v1, v2 = edge.endpoints()
        midpoint = (v1.position + v2.position) / 2.0
        
        # Perform edge collapse operation [S2]
        current_mesh.collapse_edge(edge, midpoint)
        
        # Step 6: Update 1-ring neighbors and heap [S2]
        affected_edges = current_mesh.get_1ring_edges(edge)
        
        for a_edge in affected_edges:
            new_cost = a_edge.length()
            heapq.heappush(edge_heap, (new_cost, a_edge.id))
            a_edge.set_cost(new_cost)
            
        hierarchy.append(current_mesh)
        
    return hierarchy

def hoppe_consistency_check(mesh, edge):
    # Standard Hoppe checks: edge must not be on boundary,
    # collapse must not create duplicate vertices or self-intersections [G]
    if edge.is_boundary():
        return False
    v1, v2 = edge.endpoints()
    neighbors_v1 = mesh.get_1ring_vertices(v1)
    neighbors_v2 = mesh.get_1ring_vertices(v2)
    if v1 in neighbors_v2 or v2 in neighbors_v1:
        return False
    return True
```

**Edge Cases and Pitfalls:**
*   **Self-intersections:** Mitigated by collapsing to the midpoint rather than an optimized position, which keeps computation costs low and minimizes self-intersection risk [S2].
*   **Isotropy Degradation:** Flat regions may over-collapse without the length constraint $l_{max} = 2\delta \sin(\sigma_{max})$ [S1].
*   **Degenerate Input:** The consistency checks must handle cases where edge collapse would create non-manifold geometry or duplicate vertices [S2].

## Subtopics

### [Quadric Error Metrics Simplification](16816-quadric-error-metrics-simplification.md)
Quadric Error Metrics (QEM) Simplification is a class of surface simplification algorithms that iteratively collapses edges to reduce polygon count while minimizing local geometric error. It measures the difference in discrete curvature between meshes locally, making it highly effective for piecewise linear surfaces where flat regions require fewer facets to represent the same shape. Unlike strict topology-preserving progressive meshing, standard QEM prioritizes visual fidelity and error minimization over manifold connectivity, though constrained variants exist to enforce enclosure properties.

### View-dependent Refinement of Progressive Meshes
A pioneering approach for dynamically simplifying meshes based on viewer configuration, using a series of sequential edge collapses and refinements [S5, unofficial]. Progressive meshing is difficult to compute in parallel, especially on a GPU, due to the sequential nature of edge collapses/refinements [S5, unofficial]. The method does not work with dynamic geometry because it requires a lot of precalculated edge collapses, which would need to be reworked from scratch for incremental updates [S5, unofficial]. The approach is difficult to adapt for large, streaming, procedural worlds because large scale mesh simplification is a global operation, making it hard to work with paging or chunked data [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method is dominated by heap operations, with each collapse requiring updating the heap for incident edges [S2].
*   **Memory Complexity:** Not given in the sources. Requires storage for the mesh and the edge heap [S2].
*   **Performance:** The method is computationally efficient due to the simple error metric (length only) and the avoidance of position optimization [S2].
*   **Strengths:** Guarantees topological consistency; low risk of self-intersection; simple implementation; produces even vertex distribution [S2].
*   **Limitations:** Does not preserve sharp edges or high-curvature details as effectively as quadric-based methods; geometric fidelity may degrade at very coarse resolutions compared to optimized methods [S2].
*   **When to use:** When topological integrity is critical and an even distribution of vertices is desired, such as in hierarchical isosurface segmentation for medical imaging [S2].
*   **When not to use:** When preserving sharp features or minimizing geometric error in high-curvature regions is the primary goal [S2].

## Background
The method is based on the progressive mesh representation introduced by Hugues Hoppe [S2]. It was applied in the context of hierarchical isosurface segmentation for brain cortex models, where preserving main folds (gyri and sulci) while eliminating details was required for multi-resolution mapping [S2]. The consistency checks used are from Hoppe et al. [S2].

## Sources
[S1] Feature-Preserving Offset Mesh Generation from Topology-Adapted Octrees, https://doi.org/10.1111/cgf.14906, fetched 2026-10-26
[S2] Hierarchical Isosurface Segmentation Based on Discrete Curvature, https://inria.hal.science/inria-00537470v1, fetched 2026-10-26
[S3] Multiresolution tetrahedral meshes: an analysis and a comparison, https://doi.org/10.1109/smi.2002.1003532, fetched 2026-10-26
[S4] Voxel Compression: A Detailed Overview - flyriver.com, https://www.flyriver.com/g/voxel-compression, fetched 2026-10-27 (unofficial)
[S5] July 2012 – 0 FPS, https://0fps.net/2012/07/, fetched 2026-10-30 (unofficial)
