# Snap Marching Cubes

Also called: SnapMC, Snapping for Quality Mesh Generation, UMC SnapMC

## Overview
Snap Marching Cubes (SnapMC) is an isosurface extraction algorithm that improves mesh quality by reducing the number of triangles and eliminating small angles, which are common artifacts in standard Marching Cubes [S2, unofficial]. It achieves this by modifying the scalar field to "snap" isosurface vertices that are geometrically close to grid vertices onto those grid vertices, causing nearby geometric elements to merge [S2, unofficial]. Unlike adaptive methods such as [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md) or [DelIso](954-extended-clcir-plic-reconstruction.md), SnapMC is non-adaptive and operates on a uniform grid, offering a significant speedup over adaptive counterparts at the cost of potential topological errors and non-manifold surfaces [S2, unofficial].

## How it works
SnapMC extends the standard Marching Cubes algorithm by introducing a preprocessing step that alters the scalar field values based on a snapping threshold, followed by a postprocessing step to restore geometric accuracy for vertices that were snapped [S2, unofficial].

### Inputs and Outputs
- **Input**: A scalar grid $V$ of dimensions $N_x \times N_y \times N_z$, an isovalue $\sigma$, and a snap parameter $\gamma \in [0, 0.5]$ [S2, unofficial]. The grid edge length is $L$ [S2, unofficial].
- **Output**: A triangle mesh representing the isosurface [S2, unofficial].

### Data Structures
1.  **Scalar Grid**: A 3D array storing scalar values at grid vertices [S2, unofficial].
2.  **Original Isosurface Vertices**: A list of 3D coordinates computed from the original scalar grid using standard Marching Cubes interpolation [S2, unofficial]. This is required for the final repositioning step [S2, unofficial].
3.  **Extended Lookup Table**: A Marching Cubes lookup table that handles "zero" vertices (where the scalar value equals the isovalue) [S2, unofficial]. Standard Marching Cubes tables treat zero as positive or negative; SnapMC requires a table that distinguishes $+$, $-$, and $=$ states to correctly handle vertices where the scalar value has been reset to $\sigma$ [S2, unofficial].

### Parameters
-   **$\gamma$ (gamma)**: The snap parameter, typically $0.3$ [S2, unofficial]. It controls the snapping distance threshold as $\gamma L$ [S2, unofficial]. Smaller values result in less triangle reduction and fewer topological changes [S2, unofficial].
-   **$L$**: The length of the grid edges [S2, unofficial].
-   **$\sigma$**: The isovalue [S2, unofficial].

### Algorithm Steps
1.  **Compute Original Isosurface Vertices**:
    Run standard Marching Cubes on the original scalar grid $V$ [S2, unofficial]. For every bipolar edge (an edge connecting a vertex with value $<\sigma$ to one with value $>\sigma$), compute the intersection point using linear interpolation [S2, unofficial]. Store these points in a list $P_{orig}$ [S2, unofficial].
    $$ t = \frac{\sigma - f_a}{f_b - f_a} $$
    $$ P = V_a + t(V_b - V_a) $$
    where $V_a, V_b$ are grid vertices and $f_a, f_b$ are their scalar values [S2, unofficial].

2.  **Identify Snapping Candidates**:
    For each grid vertex $v$, check if any vertex in $P_{orig}$ lies within a distance $\gamma L$ of $v$ [S2, unofficial].
    $$ \text{dist}(v, p) < \gamma L \quad \text{for some } p \in P_{orig} $$

3.  **Modify Scalar Field**:
    For every grid vertex $v$ identified in Step 2, reset its scalar value in the grid to the isovalue $\sigma$ [S2, unofficial].
    $$ V[v] \leftarrow \sigma $$
    This creates "zero" vertices in the grid [S2, unofficial].

4.  **Run Extended Marching Cubes**:
    Execute the Marching Cubes algorithm on the modified scalar grid using the extended lookup table that supports zero-valued vertices [S2, unofficial]. This generates a new set of isosurface vertices $P_{snap}$ and triangles [S2, unofficial].
    -   If an isosurface vertex $w$ lies exactly on a grid vertex $v$ (because $V[v] = \sigma$), its position is initially set to $v$ [S2, unofficial].

5.  **Reposition Snapped Vertices**:
    For each isosurface vertex $w$ in the output mesh that lies on a grid vertex $v$ (i.e., $w = v$), find the closest vertex $p \in P_{orig}$ to $v$ [S2, unofficial]. Move $w$ to the position of $p$ [S2, unofficial].
    $$ w \leftarrow \arg\min_{p \in P_{orig}} \text{dist}(v, p) $$
    This step ensures that the final mesh geometry approximates the original isosurface as closely as possible, while retaining the topological simplification (merged vertices/triangles) achieved in Step 4 [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def snap_marching_cubes(grid, isovalue, gamma, edge_length):
    """
    grid: 3D array of scalar values (Nz, Ny, Nx)
    isovalue: float sigma
    gamma: float in [0, 0.5]
    edge_length: float L
    """
    threshold = gamma * edge_length
    threshold_sq = threshold * threshold
    
    # Step 1: Compute original isosurface vertices
    # Requires a standard Marching Cubes implementation that returns
    # intersection points on bipolar edges.
    # P_orig: list of 3D points
    P_orig = compute_marching_cubes_intersections(grid, isovalue)
    
    # Step 2 & 3: Identify and modify scalar field
    modified_grid = grid.copy()
    snapped_indices = []
    
    # Iterate over internal grid vertices (skip boundaries to avoid edge cases)
    for z in range(1, grid.shape[0] - 1):
        for y in range(1, grid.shape[1] - 1):
            for x in range(1, grid.shape[2] - 1):
                v_pos = (x, y, z)
                v_val = grid[z, y, x]
                
                # Check if any original isosurface vertex is close to this grid vertex
                is_snapped = False
                for p in P_orig:
                    dx = p[0] - v_pos[0]
                    dy = p[1] - v_pos[1]
                    dz = p[2] - v_pos[2]
                    if dx*dx + dy*dy + dz*dz < threshold_sq:
                        is_snapped = True
                        break
                
                if is_snapped:
                    modified_grid[z, y, x] = isovalue
                    snapped_indices.append((z, y, x))
                    
    # Step 4: Run Extended Marching Cubes
    # Assumes a standard MC implementation that supports '=' (zero) cases
    # Returns a mesh with vertices and triangles
    mesh_snap = extended_marching_cubes(modified_grid, isovalue)
    
    # Step 5: Reposition snapped vertices
    # Map from grid vertex index to closest original isosurface vertex
    snap_map = {}
    for v_idx in snapped_indices:
        v_pos = v_idx
        closest_p = None
        min_dist_sq = float('inf')
        for p in P_orig:
            dx = p[0] - v_pos[0]
            dy = p[1] - v_pos[1]
            dz = p[2] - v_pos[2]
            d_sq = dx*dx + dy*dy + dz*dz
            if d_sq < min_dist_sq:
                min_dist_sq = d_sq
                closest_p = p
        snap_map[v_idx] = closest_p
        
    # Update mesh vertices that correspond to snapped grid vertices
    for vertex in mesh_snap.vertices:
        if vertex.grid_vertex_index in snap_map:
            vertex.position = snap_map[vertex.grid_vertex_index]
            
    return mesh_snap
```

### Edge Cases and Pitfalls
-   **Topological Changes**: Snapping merges geometrically close vertices even if they are not connected by edges. This can eliminate small components, join separate components, or close loops in the isosurface [S2, unofficial].
-   **Non-Manifold Surfaces**: Because snapping merges vertices that may not share an edge, the resulting surface often becomes non-manifold. Facets glued together appear as duplicate triangles with opposite orientation [S2, unofficial]. This is problematic for simulation software which typically requires manifold meshes.
-   **Precision**: The snapping distance $\gamma L$ must be chosen carefully. If $\gamma$ is too large, excessive topological errors occur. If too small, the triangle reduction benefit is minimal [S2, unofficial].
-   **Lookup Table**: Standard Marching Cubes lookup tables do not handle zero-valued vertices correctly. An extended lookup table distinguishing $+$, $-$, and $=$ states is required [S2, unofficial].
-   **Boundary Handling**: Grid vertices on chunk or domain boundaries should be skipped during the snapping check to avoid out-of-bounds memory access and incorrect edge crossing calculations [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: SnapMC takes approximately twice as long as standard Marching Cubes due to the extra time spent in snapping scalar values and the repositioning step [S2, unofficial]. It runs an order of magnitude faster than adaptive methods like [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md) and [DelIso](954-extended-clcir-plic-reconstruction.md) [S2, unofficial].
-   **Memory Complexity**: Requires storage for the original isosurface vertices ($P_{orig}$) and the modified scalar grid. This is $O(N)$ where $N$ is the number of grid vertices [S2, unofficial].
-   **Performance**:
    -   On a $256 \times 256 \times 256$ aneurism dataset ($\gamma=0.3$), SnapMC produced 106,356 triangles in 1.52s, reducing triangle count by ~39% compared to Regular MC (175,832 triangles, 0.71s) [S2, unofficial].
    -   On a $256 \times 256 \times 256$ bonsai dataset ($\gamma=0.3$), SnapMC produced 729,623 triangles in 2.19s, reducing triangle count by ~43% compared to Regular MC (1,284,542 triangles) [S2, unofficial].
    -   On the lobster dataset ($\gamma=0.3$), SnapMC produced 223K triangles in 0.8s with a minimum angle of 13.3 degrees [S2, unofficial].
-   **Mesh Quality**: SnapMC produces well-shaped triangles suitable for numerical simulation, with minimum angles typically above 13 degrees for $\gamma=0.3$ [S2, unofficial]. In contrast, DelIso often produces degenerate triangles with zero area or edge length and minimum angles of 0 degrees [S2, unofficial].
-   **Accuracy**: The directed Hausdorff distance from the SnapMC surface to the original surface is less than 1 for various datasets (e.g., 0.86 for the lobster dataset) [S2, unofficial].
-   **Strengths**: Significant triangle reduction (25-40%), improved minimum triangle angles, fast execution compared to adaptive methods [S2, unofficial].
-   **Limitations**: Produces non-manifold surfaces, potential topological errors, not adaptive (does not add detail in high-curvature areas) [S2, unofficial].

## Background
Snap Marching Cubes was introduced by S. Raman and R. Wenger in the paper "Quality Isosurface Mesh Generation" [S2, unofficial]. The algorithm was developed to address the notorious issue of Marching Cubes producing isosurface triangles with small angles, which occur when two triangle vertices are very close to a grid vertex while the third is far away [S2, unofficial]. The method modifies the scalar field to snap these close vertices onto grid vertices, thereby merging geometrically close elements [S2, unofficial]. The source code and executables for SnapMC are publicly available at www.cse.ohio-state.edu/graphics/isotable [S2, unofficial].

## Sources
[S1] Lin20/PushingVoxelsForward, https://github.com/Lin20/PushingVoxelsForward, fetched 2026-10-27 (unofficial)
[S2] Extended Marching Cubes, http://www.cse.ohio-state.edu/~wenger/publications/isomesh.pdf, fetched 2026-10-29 (unofficial)
