# Sample-based Simplification
Also called: HAVS-based Simplification

## Overview
Sample-based simplification is a method for reducing the geometric complexity of unstructured tetrahedral meshes during volume rendering by selectively removing triangle primitives rather than resampling the domain [S1, S2, unofficial]. It differs from domain-based simplification (which uses edge-collapses to resample the domain) by operating directly on the rendered primitives, maintaining the original mesh topology while discarding less important triangles [S1, S2, unofficial]. This approach is designed for interactive rendering of large datasets, particularly within the HAVS volume rendering system [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh where each tetrahedron is represented by its triangular faces. Each triangle has an associated importance value.
*   **Output:** A subset of the original triangle indices to be rendered, determined by a dynamic threshold.

**Data Structures:**
1.  **Triangle List:** A list of all triangle indices from the tetrahedral mesh.
2.  **Importance Scores:** A precomputed scalar value for each triangle, determining its visual significance.
3.  **Dynamic Threshold:** A runtime parameter controlling the number of internal triangles to render.

**Numbered Steps:**
1.  **Preprocessing (Importance Calculation):** Compute an importance heuristic for every triangle in the mesh. The specific heuristics are not detailed in the provided sources, but they are calculated once before interaction [S1, S2, unofficial].
2.  **Sorting:** Sort the triangle indices in descending order based on their precomputed importance scores [S1, S2, unofficial].
3.  **Runtime Selection:**
    *   Identify all **boundary triangles** of the mesh. These are always included in the render set [S1, S2, unofficial].
    *   Select a dynamically adjusted number of **internal triangles** from the sorted list, starting from the highest importance [S1, S2, unofficial].
    *   The dynamic adjustment is typically driven by frame rate or user interaction constraints to maintain interactivity [S1, S2, unofficial].
4.  **Rendering:** Render the union of the boundary triangles and the selected internal triangles using the HAVS volume rendering system [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase
def preprocess(mesh):
    """
    mesh: Tetrahedral mesh object containing triangles and field values.
    Returns:
        sorted_indices: List of triangle indices sorted by importance (descending).
        boundary_indices: Set of triangle indices that are on the mesh boundary.
    """
    triangles = mesh.get_all_triangles()
    importance_scores = []
    
    # Calculate importance for each triangle using heuristics [S1, S2, unofficial]
    # Note: Specific heuristic formulas are not provided in the sources [G]
    for i, tri in enumerate(triangles):
        score = compute_importance_heuristic(tri, mesh)
        importance_scores.append((score, i))
    
    # Sort by importance descending
    importance_scores.sort(key=lambda x: x[0], reverse=True)
    sorted_indices = [idx for score, idx in importance_scores]
    
    # Identify boundary triangles
    boundary_indices = set(mesh.get_boundary_triangles())
    
    return sorted_indices, boundary_indices

# Runtime Phase
def render_step(sorted_indices, boundary_indices, max_internal_triangles):
    """
    sorted_indices: Pre-sorted list of triangle indices.
    boundary_indices: Set of boundary triangle indices.
    max_internal_triangles: Dynamic limit on internal triangles to render.
    Returns:
        render_set: List of triangle indices to draw.
    """
    render_set = set(boundary_indices)
    
    # Add top internal triangles up to the dynamic limit
    internal_count = 0
    for idx in sorted_indices:
        if idx not in boundary_indices:
            if internal_count < max_internal_triangles:
                render_set.add(idx)
                internal_count += 1
            else:
                break
                
    return list(render_set)
```

**Parameters:**
*   **max_internal_triangles:** The maximum number of non-boundary triangles to include. This value is adjusted dynamically during interaction [S1, S2, unofficial]. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Definition:** The definition of "boundary triangles" must be consistent with the mesh topology. If the mesh is closed, this set may be empty or require specific handling [G].
*   **Dynamic Adjustment:** The method relies on a heuristic to adjust `max_internal_triangles` in real-time. If the adjustment is too slow, frame rates may drop; if too fast, visual quality may fluctuate [S1, S2, unofficial].
*   **Preprocessing Cost:** Calculating importance for all triangles upfront can be expensive for very large meshes, but this is a one-time cost [S1, S2, unofficial].

## Subtopics

### HAVS Volume Rendering
The underlying rendering system that operates on triangle faces of tetrahedral meshes. It provides the context for why triangle-level simplification is effective. [HAVS](1000-havs-volume-rendering.md)

### Domain-based Simplification
The alternative approach that resamples the domain using edge-collapses. Sample-based simplification is distinct because it removes primitives directly without modifying the underlying domain structure. [Domain-based](1032-mesh-simplification.md)

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N \log N)$ due to sorting $N$ triangles, where $N$ is the number of triangles. Runtime selection is $O(1)$ if the threshold is precomputed, or $O(K)$ where $K$ is the number of selected triangles.
*   **Memory Complexity:** Requires storing the importance scores for all triangles ($O(N)$) and the sorted index list ($O(N)$).
*   **Performance:** Designed for interactive rendering of large datasets [S1, S2, unofficial]. Performance depends on the efficiency of the importance heuristic and the dynamic adjustment mechanism.
*   **Strengths:** Simple implementation; does not require modifying the mesh topology; suitable for dynamic data where topology might change [S1, S2, unofficial].
*   **Limitations:** Visual quality may degrade if important internal triangles are excluded; relies on the accuracy of the importance heuristic [S1, S2, unofficial].
*   **When to Use:** Interactive visualization of large unstructured tetrahedral meshes where frame rate is critical and the mesh topology is static or changes slowly [S1, S2, unofficial].
*   **When Not to Use:** When topological changes are frequent and complex, or when exact geometric fidelity is required over all regions [S1, S2, unofficial].

## Background
Sample-based simplification was introduced by Callahan et al. as an alternative to domain-based simplification for volume rendering of unstructured grids [S1, S2, unofficial]. It is based on the HAVS (Hardware-Accelerated Volume Rendering System) which operates on triangle faces of tetrahedral meshes [S1, S2, unofficial]. The method was developed to address the challenges of visualizing large datasets interactively, particularly in the context of time-varying and dynamic data [S1, S2, unofficial].

## Sources
[S1] A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
