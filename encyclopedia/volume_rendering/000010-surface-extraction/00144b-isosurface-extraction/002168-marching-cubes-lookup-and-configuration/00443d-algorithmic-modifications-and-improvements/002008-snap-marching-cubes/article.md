# Snap Marching Cubes

Also called: SnapMC, Snapping for Quality Mesh Generation, UMC SnapMC

## Overview
Snap Marching Cubes (SnapMC) is an isosurface extraction algorithm that improves mesh quality by reducing the number of triangles and eliminating small, noisy angles [S2, unofficial]. It achieves this by modifying the scalar field: if an isosurface vertex is sufficiently close to a grid vertex, the grid vertex's scalar value is snapped to the isovalue, causing the isosurface vertex to merge with the grid vertex [S1, S2, unofficial]. This reduces vertex count by approximately 25–40% compared to standard Marching Cubes [S2, unofficial]. Unlike adaptive methods such as Afront or DelIso, SnapMC is non-adaptive and runs significantly faster, though it may produce non-manifold surfaces due to topological changes [S2, unofficial].

## How it works
SnapMC builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) by using an extended lookup table that distinguishes vertices with scalar values equal to the isovalue ('=' case) [S2, unofficial]. The algorithm operates in two phases: a pre-processing phase to identify snapping candidates and a marching phase on the modified field, followed by a post-processing repositioning step [S2, unofficial].

### Inputs and Outputs
*   **Input:** A structured 3D scalar grid $V$ of dimensions $(D_x+1) \times (D_y+1) \times (D_z+1)$, an isovalue $\sigma$, and a snap parameter $\gamma \in [0, 0.5]$ [S2, unofficial].
*   **Output:** A triangle mesh representing the isosurface $\{x \mid V(x) = \sigma\}$ [S2, unofficial].

### Data Structures
The implementation uses a chunk-based structure for parallel processing. A `UMC_Chunk` contains:
*   `pem`: Integer parameter (context-specific) [S1, unofficial].
*   `dim`: `uint32_t` defining the local grid dimension size [S1, unofficial].
*   `snap_threshold`: `float` defining the maximum distance for snapping [S1, unofficial].
*   `grid_signs`: `uint16_t*` pointer to the sign bits of the grid vertices [S1, unofficial].
*   `grid_verts`: `struct UMC_Isovertex*` pointer to the grid vertex data [S1, unofficial].
*   `edges`: `struct UMC_Edge*` pointer to edge data [S1, unofficial].
*   `snapped_count`: `uint32_t` counter for snapped vertices [S1, unofficial].

The `UMC_Isovertex` structure holds position and index information [S1, unofficial]. The `UMC_Edge` structure tracks edge state, including a `snapped` flag [S1, unofficial].

### Parameters
*   $\gamma$: Snap parameter controlling the threshold. Typical value: $0.3$ [S2, unofficial].
*   $L$: Length of grid edges (assumed uniform) [S2, unofficial].
*   Snap Threshold: $\gamma L$. If distance $< \gamma L$, snapping occurs [S2, unofficial].

### Algorithm Steps

1.  **Identify Bipolar Edges and Initial Vertices:**
    Iterate over all cells in the grid. For each cell, determine bipolar edges (edges connecting vertices with signs on opposite sides of the isovalue). For each bipolar edge, compute the isosurface vertex position $w$ using linear interpolation [S2, unofficial]. Store these positions in a temporary list `out_indexes` [S1, unofficial].

2.  **Snap Scalar Values:**
    Iterate through each grid vertex $v$ identified in Step 1 as having a nearby isosurface vertex.
    *   Calculate the distance $d$ from the interpolated isosurface vertex $w$ to the grid vertex $v$.
    *   If $d < \gamma L$, set the scalar value of grid vertex $v$ to the isovalue $\sigma$ [S2, unofficial].
    
    *Note: The source [S1, unofficial] describes a bitwise implementation for a specific parallel chunk-based variant. It calculates grid coordinates $(x,y,z)$ from an index, checks six surrounding edges for the closest edge crossing, and if the distance is below a threshold, updates `grid_signs` by clearing and setting specific bits to represent the '=' case, then marks edges as snapped. This bitwise logic is a specific optimization for the lookup table generation and should be treated as an implementation detail of the sign propagation rather than the core geometric snapping definition provided by [S2, unofficial]. The core geometric definition is resetting the scalar value.*

3.  **Run Marching Cubes:**
    Execute the standard Marching Cubes algorithm (using the extended lookup table from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md)) on the modified scalar grid [S2, unofficial]. This produces a mesh where vertices snapped to grid vertices are located exactly at those grid coordinates.

4.  **Reposition Snapped Vertices:**
    For each vertex $w$ in the output mesh that lies exactly on a grid vertex $v$ (due to snapping):
    *   Find the closest isosurface vertex $w_{orig}$ from the original interpolation in Step 1 that was associated with $v$.
    *   Move $w$ to the position of $w_{orig}$ [S2, unofficial].
    
    This step restores the geometric accuracy of the surface while retaining the topological simplification (merged vertices) achieved by the snap.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def snap_marching_cubes(grid, isovalue, gamma):
    """
    grid: 3D array of scalar values, shape (Nx+1, Ny+1, Nz+1)
    isovalue: float sigma
    gamma: float in [0, 0.5]
    L: float, length of grid edges (assumed 1.0 for normalized grids)
    """
    L = 1.0
    threshold = gamma * L
    
    # Step 1: Identify isosurface vertices and their associated grid vertices
    # Store pairs (grid_vertex_index, interpolated_position)
    candidates = []
    
    for each cell in grid:
        # Determine signs of 8 vertices relative to isovalue
        signs = [1 if v > isovalue else (-1 if v < isovalue else 0) for v in cell.vertices]
        
        # Find bipolar edges (signs differ)
        for edge in cell.bipolar_edges:
            # Linear interpolation to find isosurface vertex w
            w = interpolate_isosurface(edge, isovalue)
            
            # Identify the two grid vertices of this edge
            v1, v2 = edge.vertices
            
            # Check distance to both
            d1 = distance(w, v1)
            d2 = distance(w, v2)
            
            if d1 < threshold:
                candidates.append((v1.index, w))
            if d2 < threshold:
                candidates.append((v2.index, w))
                
    # Step 2: Snap scalar values
    # Create a copy of the grid to modify
    modified_grid = copy(grid)
    
    # Use a set to avoid redundant checks for the same grid vertex
    snapped_vertices = {}
    
    for v_index, w in candidates:
        if v_index not in snapped_vertices:
            snapped_vertices[v_index] = w
            
    for v_index, w in snapped_vertices.items():
        # Reset scalar value to isovalue
        modified_grid[v_index] = isovalue
        
    # Step 3: Run Marching Cubes on modified grid
    # Uses extended lookup table handling '=' case
    mesh = marching_cubes_extended(modified_grid, isovalue)
    
    # Step 4: Reposition vertices
    for triangle in mesh.triangles:
        for vertex in triangle.vertices:
            if vertex.is_on_grid_vertex():
                # Find the original interpolated position for this grid vertex
                if vertex.grid_index in snapped_vertices:
                    original_w = snapped_vertices[vertex.grid_index]
                    vertex.position = original_w
                    
    return mesh
```

### Edge Cases and Pitfalls
*   **Topological Changes:** Snapping merges geometrically close vertices even if they are not connected by edges. This can eliminate small components, join separate components, or close loops in the isosurface [S2, unofficial].
*   **Non-Manifold Surfaces:** Merging vertices that do not share an edge often produces non-manifold surfaces. Facets glued together appear as duplicate triangles with opposite orientation [S2, unofficial]. This is problematic for simulation software [S2, unofficial].
*   **Precision:** The snapping threshold $\gamma L$ must be chosen carefully. Too large a $\gamma$ causes excessive topological errors; too small a $\gamma$ yields minimal triangle reduction [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** SnapMC takes approximately twice as long as standard Marching Cubes due to the extra time spent snapping scalar values and repositioning vertices [S2, unofficial].
*   **Memory Complexity:** Requires storage for the scalar grid, the modified grid (or in-place modification), and temporary storage for candidate vertices [S2, unofficial].
*   **Performance:**
    *   On a $256^3$ aneurism dataset ($\gamma=0.3$): 106,356 triangles in 1.52s (vs 175,832 triangles in 0.71s for Regular MC) [S2, unofficial].
    *   On a $256^3$ bonsai dataset ($\gamma=0.3$): 729,623 triangles in 2.19s (vs 1,284,542 triangles for Regular MC) [S2, unofficial].
    *   On a lobster dataset ($\gamma=0.3$): 223K triangles in 0.8s [S2, unofficial].
*   **Mesh Quality:**
    *   Reduces triangle count by 25–40% [S2, unofficial].
    *   Improves minimum triangle angles (e.g., min angle 13.3° on lobster vs 0° for DelIso) [S2, unofficial].
    *   Directed Hausdorff distance to original surface is typically < 1 [S2, unofficial].
*   **Strengths:** Produces well-shaped triangles suitable for numerical simulation; significantly faster than adaptive methods like Afront and DelIso; reduces mesh size [S2, unofficial].
*   **Limitations:** Produces non-manifold surfaces; may alter topology; not adaptive [S2, unofficial].
*   **When to Use:** When mesh size reduction and triangle quality are prioritized over strict manifoldness, or as a preprocessing step for simulation where non-manifold artifacts can be handled or removed [S2, unofficial].
*   **When Not to Use:** When a strictly manifold surface is required for downstream applications like finite element analysis without post-processing [S2, unofficial].

## Background
SnapMC was introduced by S. Raman and R. Wenger in the paper "Quality Isosurface Mesh Generation" (2008) [S2, unofficial]. It was developed to address the issue of Marching Cubes producing triangles with small angles, which are detrimental to numerical simulations [S2, unofficial]. The algorithm is part of the Extended Marching Cubes family, utilizing the extended lookup table to handle the '=' case [S2, unofficial]. Source code and executables were made publicly available at www.cse.ohio-state.edu/graphics/isotable [S2, unofficial].

## Sources

[S1] Lin20/PushingVoxelsForward, https://github.com/Lin20/PushingVoxelsForward, fetched 2026-10-27 (unofficial)
[S2] Extended Marching Cubes, http://www.cse.ohio-state.edu/~wenger/publications/isomesh.pdf, fetched 2026-10-29 (unofficial)
