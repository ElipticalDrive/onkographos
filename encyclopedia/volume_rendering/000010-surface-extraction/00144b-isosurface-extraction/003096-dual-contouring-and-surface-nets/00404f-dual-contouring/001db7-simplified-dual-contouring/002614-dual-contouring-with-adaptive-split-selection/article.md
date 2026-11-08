# Dual Contouring with Adaptive Split Selection


## Overview
Dual Contouring with Adaptive Split Selection is a mesh extraction method that converts voxel grid data into a triangle mesh by constructing quads from connected voxels and splitting them into triangles based on geometric alignment or weight products [S1, unofficial].

## How it works
**Inputs:**
- `coords`: Tensor of coordinates for active voxels [S1, unofficial].
- `dual_vertices`: Tensor of dual vertex offsets [S1, unofficial].
- `grid_size`: 1D tensor of size 3, specifying the voxel grid dimensions [S1, unofficial].
- `split_weight`: Optional tensor used to guide quad splitting in non-training mode or drive mid-vertex interpolation in training mode [S1, unofficial].
- `train`: Boolean flag determining whether to use the training-specific split logic with mid-vertex interpolation [S1, unofficial].
- `N`: Number of dual vertices, used in vertex coordinate normalization [S1, unofficial].

**Outputs:**
- `mesh_vertices`: Tensor of extracted mesh vertex coordinates [S1, unofficial].
- `mesh_triangles`: Tensor of shape (-1, 3) containing vertex indices for each triangle [S1, unofficial].

**Data Structures:**
- `quad_indices`: Tensor of shape (L, 4) containing vertex indices for each quad [S1, unofficial].
- `active_voxel_map`: Hashmap using 3D indices as values to store active voxels for fast neighbor lookup [S1, unofficial].

**Steps:**

1.  **Vertex Coordinate Normalization**: Compute the initial mesh vertices using the formula:
    $$ \text{mesh\_vertices} = \frac{\text{coords.float()} + \text{dual\_vertices}}{2 \times N} - 0.5 $$
    [S1, unofficial]

2.  **Active Voxel Storage**: Store active voxel coordinates into a hashmap (`active_voxel_map`) using their 3D indices as values [S1, unofficial].

3.  **Quad Construction**:
    a.  Find connected voxels by adding edge neighbor offsets to the coordinates:
        $$ \text{edge\_neighbor\_voxel} = \text{coords.reshape}(N, 1, 1, 3) + \text{edge\_neighbor\_voxel\_offset} $$
        [S1, unofficial]
    b.  Lookup connected voxel indices in `active_voxel_map` [S1, unofficial].
    c.  Filter out invalid entries (specifically `0xFFFFFFFF`) to obtain valid quad indices, resulting in `quad_indices` of shape (L, 4) [S1, unofficial].

4.  **Split Selection (Non-Training Mode)**:
    *   **Case A: `split_weight` is None**:
        a.  Calculate two potential triangle splits for each quad [S1, unofficial].
        b.  Compute normals for both splits [S1, unofficial].
        c.  Measure alignment using the dot product of normals:
            $$ \text{align} = |\text{normals}_0 \cdot \text{normals}_1| $$
            [S1, unofficial]
        d.  Select the split with higher alignment (indicating a flatter quad) using:
            $$ \text{split\_choice} = \text{torch.where}(\text{align}_0 > \text{align}_1, \dots) $$
            [S1, unofficial]
    *   **Case B: `split_weight` is provided**:
        a.  Calculate split weights for vertices (0, 2) and (1, 3):
            $$ w_{02} = \text{split\_weight\_ws}[:, 0] \times \text{split\_weight\_ws}[:, 2] $$
            $$ w_{13} = \text{split\_weight\_ws}[:, 1] \times \text{split\_weight\_ws}[:, 3] $$
            [S1, unofficial]
        b.  Select the split based on which product is larger [S1, unofficial].

5.  **Split Selection (Training Mode)**:
    *   **Constraint**: `split_weight` must be provided; otherwise, an assertion error is raised [S1, unofficial].
    a.  Compute mid-vertices for each quad by interpolating between the mean of vertices (0, 2) and the mean of vertices (1, 3), weighted by their respective weight products:
        $$ \text{mid\_vertices} = \frac{w_{02} \times \text{mean\_v02} + w_{13} \times \text{mean\_v13}}{w_{02} + w_{13}} $$
        [S1, unofficial]
    b.  Append `mid_vertices` to `mesh_vertices` [S1, unofficial].
    c.  Update `quad_indices` to include these new vertices [S1, unofficial].
    d.  Form triangles using a specific training split configuration that incorporates the mid-vertices [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def cross(a, b):
    return [
        a[1]*b[2] - a[2]*b[1],
        a[2]*b[0] - a[0]*b[2],
        a[0]*b[1] - a[1]*b[0]
    ]

def dual_contouring_adaptive_split(coords, dual_vertices, grid_size, split_weight=None, train=False, N=None):
    """
    Extracts a triangle mesh from voxel data using adaptive quad splitting.
    
    Args:
        coords: List of [x, y, z] coordinates for active voxels.
        dual_vertices: List of [x, y, z] dual vertex offsets.
        grid_size: List of 3 integers specifying voxel grid dimensions.
        split_weight: Optional list of [w0, w1, w2, w3] guiding splits.
        train: Boolean flag for training mode.
        N: Number of dual vertices. If None, inferred from coords.
        
    Returns:
        mesh_vertices: List of vertex coordinates.
        mesh_triangles: List of triangle index triplets.
    """
    if N is None:
        N = len(coords)
        
    # Step 1: Normalize vertex coordinates
    mesh_vertices = [
        [(c[i] + d[i]) / (2 * N) - 0.5 for i in range(3)]
        for c, d in zip(coords, dual_vertices)
    ]
    
    # Step 2: Store active voxels in a hashmap for neighbor lookup
    active_voxel_map = {tuple(c): i for i, c in enumerate(coords)}
    
    # Step 3: Find connected voxels and build quads
    edge_neighbor_offsets = [(1, 0, 0), (0, 1, 0), (0, 0, 1)]
    quad_indices = []
    INVALID_IDX = 0xFFFFFFFF
    
    for i, coord in enumerate(coords):
        neighbors = []
        for offset in edge_neighbor_offsets:
            neighbor_coord = tuple(coord[j] + offset[j] for j in range(3))
            if neighbor_coord in active_voxel_map:
                neighbors.append(active_voxel_map[neighbor_coord])
            else:
                neighbors.append(INVALID_IDX)
        # Filter out invalid entries to get valid quad indices
        valid_neighbors = [n for n in neighbors if n != INVALID_IDX]
        if len(valid_neighbors) >= 3:
            quad_indices.append([i] + valid_neighbors[:3])
            
    # Step 4 & 5: Split Selection
    mesh_triangles = []
    L = len(quad_indices)
    
    if not train:
        if split_weight is None:
            # Geometric alignment based split
            for i in range(L):
                q = quad_indices[i]
                v0, v1, v2, v3 = [mesh_vertices[idx] for idx in q]
                # Split 0: (v0, v1, v2) and (v0, v2, v3)
                n0_a = cross([v1[j]-v0[j] for j in range(3)], [v2[j]-v0[j] for j in range(3)])
                n0_b = cross([v2[j]-v0[j] for j in range(3)], [v3[j]-v0[j] for j in range(3)])
                # Split 1: (v0, v1, v3) and (v1, v2, v3)
                n1_a = cross([v1[j]-v0[j] for j in range(3)], [v3[j]-v0[j] for j in range(3)])
                n1_b = cross([v2[j]-v1[j] for j in range(3)], [v3[j]-v1[j] for j in range(3)])
                # Alignment metric: align = (normals0 * normals1).sum().abs()
                align = abs(dot(n0_a, n1_a)) + abs(dot(n0_b, n1_b))
                # Select split with higher alignment
                if align > 0:
                    mesh_triangles.append([q[0], q[1], q[2]])
                    mesh_triangles.append([q[0], q[2], q[3]])
                else:
                    mesh_triangles.append([q[0], q[1], q[3]])
                    mesh_triangles.append([q[1], q[2], q[3]])
        else:
            # Weight-based split
            for i in range(L):
                q = quad_indices[i]
                w = split_weight[i]
                w02 = w[0] * w[2]
                w13 = w[1] * w[3]
                if w02 > w13:
                    mesh_triangles.append([q[0], q[1], q[2]])
                    mesh_triangles.append([q[0], q[2], q[3]])
                else:
                    mesh_triangles.append([q[0], q[1], q[3]])
                    mesh_triangles.append([q[1], q[2], q[3]])
    else:
        # Training mode
        assert split_weight is not None, "split_weight must be provided in training mode"
        mid_vertices = []
        for i in range(L):
            q = quad_indices[i]
            v0, v1, v2, v3 = [mesh_vertices[idx] for idx in q]
            w = split_weight[i]
            w02 = w[0] * w[2]
            w13 = w[1] * w[3]
            mean_v02 = [(v0[j] + v2[j]) / 2 for j in range(3)]
            mean_v13 = [(v1[j] + v3[j]) / 2 for j in range(3)]
            total_w = w02 + w13
            # Add epsilon to avoid division by zero [G]
            mid = [(w02 * mean_v02[j] + w13 * mean_v13[j]) / (total_w + 1e-8) for j in range(3)]
            mid_vertices.append(mid)
            
        new_vertex_start = len(mesh_vertices)
        mesh_vertices.extend(mid_vertices)
        
        for i in range(L):
            mid_idx = new_vertex_start + i
            q = quad_indices[i]
            # Form triangles using a specific training split configuration
            mesh_triangles.append([q[0], q[1], mid_idx])
            mesh_triangles.append([q[1], q[2], mid_idx])
            mesh_triangles.append([q[2], q[3], mid_idx])
            mesh_triangles.append([q[3], q[0], mid_idx])
            
    return mesh_vertices, mesh_triangles
```

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Requires storage for `mesh_vertices`, `quad_indices`, and the `active_voxel_map` [S1, unofficial].
- **Strengths**: Adapts to surface geometry by minimizing distortion in non-training mode or respecting semantic weights in training mode [S1, unofficial].
- **Limitations**: Requires careful handling of invalid voxel indices (`0xFFFFFFFF`) [S1, unofficial]. In training mode, it mandates `split_weight`, adding a dependency on external weight computation [S1, unofficial].
- **When to use**: When surface fidelity is critical and geometric alignment or user-defined weights can guide the triangulation [S1, unofficial].
- **When not to use**: Not given in the sources.

## Background
This method is part of the remeshing pipeline in the `o-voxel-gpu` project, which focuses on differentiable voxel-based rendering and mesh extraction [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
