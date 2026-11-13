# DCx (Dual Contouring over Expanded Cubes)

Also called: Dual Contouring over Expanded Cubes

## Overview
DCx is an isosurface extraction method designed to reconstruct zero-level sets from neural Unsigned Distance Functions (UDFs), addressing the lack of sign information inherent in unsigned fields [S1]. Unlike standard Dual Contouring which relies on sign changes to identify active cubes and connect vertices, DCx uses an optimization-based strategy for active cube detection and operates on 2x2x2 voxel blocks (expanded cubes) to robustly handle sparse, irregular active cube distributions and non-manifold structures [S1]. It produces single-layer, intersection-free meshes for both manifold and non-manifold inputs without requiring supervision, avoiding the double-layer entanglement and topological errors common in prior UDF extraction methods [S1].

## How it works
**Inputs:**
- A neural Unsigned Distance Function (UDF) $u(\mathbf{x})$ defined on a 3D grid.
- Grid resolution $N$ (e.g., 256).
- Thresholds $\tau_0$ and $\tau_1$ for active cube detection and supplementary sampling.

**Outputs:**
- A single-layer triangle mesh representing the zero-level set of the UDF.

**Data Structures:**
- **Grid:** A 3D array of size $N \times N \times N$ storing UDF values.
- **Expanded Cube:** A 2x2x2 block of voxels. Adjacent expanded cubes overlap, sharing 2x2x1 voxels (4 voxels in common).
- **V2M Lookup Table:** A mapping from the binary configuration of 8 voxels (active/inactive) within an expanded cube to the corresponding mesh connectivity (faces/edges) on the dual cube vertices.
- **Active Cube Set:** A set of cube indices identified as active based on the optimization step.
- **Point Set:** A set of 3D points $\mathbf{p}$ representing projected centroids.

**Parameters:**
- Learning rate for vector Adam optimizer: 0.01 [S1].
- Number of iterations for vector Adam optimizer: 200 [S1].
- Threshold for initial point selection: $\tau_0 = 1.5/N$ [S1].
- Threshold for supplementary sampling: $\tau_1 = 0.5/N$ [S1].

**Steps:**

1.  **Sampling and Optimization (Active Cube Identification):**
    - Initialize a set of candidate points $\mathcal{P}_{init}$ by selecting voxel centroids $\mathbf{c}_i$ where the UDF value $u(\mathbf{c}_i) \le \tau_0$ [S1].
    - For each point $\mathbf{p}_i \in \mathcal{P}_{init}$, perform direction-constrained optimization to project it toward the zero-level set. The optimization minimizes the UDF value along the gradient direction:
      $$ \min_{s_i} u(\mathbf{p}_i + s_i \nabla u(\mathbf{p}_i)) $$
      where $s_i$ is a scalar step size [S1].
    - Use the vector Adam optimizer with learning rate 0.01 for 200 iterations to solve for $s_i$ [S1].
    - Let $\mathbf{p}_i^{opt} = \mathbf{p}_i + s_i \nabla u(\mathbf{p}_i)$ be the optimized point.
    - Identify **Active Cubes**: A cube is marked as active if its centroid's optimized point $\mathbf{p}_i^{opt}$ lies within the cube boundaries [S1]. This defines active cubes based on occupancy, naturally confining vertices and eliminating the need for corrective clamping [S1].

2.  **Supplementary Sampling:**
    - To improve completeness in under-sampled regions, perform supplementary sampling. Select additional voxel centroids where $u(\mathbf{c}_j) < \tau_1$ [S1].
    - Apply the same direction-constrained optimization (vector Adam) to these points.
    - Mark cubes containing these optimized points as active if they satisfy the occupancy condition [S1].

3.  **Thinning (Noise and Artifact Removal):**
    - Iterate over all expanded cubes classified as Case 8 (a specific topological configuration in the V2M table).
    - **Condition 1:** If a voxel in a Case 8 configuration fails to meet structural requirements for Case 8 validity, delete that voxel. Repeat until no further voxels are removed [S1].
    - **Condition 2:** For every active cube and its 3x3x3 neighborhood, remove a redundant voxel if all edges connected to the active voxel are boundary edges and at least one endpoint is on a non-manifold edge. Repeat until no such voxels are removed [S1].
    - This step removes noise, eliminates redundant non-manifold artifacts, and mitigates holes caused by local configurations violating Case 8 rules [S1].

4.  **Meshing via V2M Lookup Table:**
    - Examine all expanded cubes spanning 2x2x2 voxels [S1].
    - For each expanded cube, determine the binary configuration of its 8 voxels (active or inactive) [S1].
    - Use the V2M Lookup Table to retrieve the predefined mesh connectivity pattern for that configuration [S1].
    - Generate faces by connecting the active vertices (corresponding to the active cubes) of the dual cube (treating the 8 cubes as 8 vertices) according to the lookup table [S1].
    - Aggregate the case IDs of adjacent expanded cubes to achieve a polygonization criterion with an effective receptive field of 4x4x4, ensuring consistency across overlapping regions [S1].

5.  **Post-processing:**
    - Apply post-processing to efficiently remove residual redundant non-manifold elements and enhance overall mesh quality [S1].
    - (Optional) Refine the extracted mesh using the optimization procedure from DCUDF to decrease Chamfer Distance, if the input UDF is not ground-truth [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dcx_extract_mesh(udf_grid, N, tau_0, tau_1, lr=0.01, iterations=200):
    """
    Extracts a mesh from a neural UDF using DCx.
    
    Args:
        udf_grid: 3D array of UDF values, shape (N, N, N).
        N: Grid resolution.
        tau_0: Threshold for initial active cube detection.
        tau_1: Threshold for supplementary sampling.
        lr: Learning rate for Adam optimizer.
        iterations: Number of optimization iterations.
        
    Returns:
        vertices: Array of vertex positions.
        faces: Array of face indices.
    """
    
    # Step 1: Initial Sampling
    # Find voxel centroids where UDF <= tau_0
    initial_points = []
    for i in range(N):
        for j in range(N):
            for k in range(N):
                if udf_grid[i, j, k] <= tau_0:
                    # Centroid of voxel (i,j,k)
                    p = np.array([i + 0.5, j + 0.5, k + 0.5])
                    initial_points.append(p)
    
    # Step 2: Direction-Constrained Optimization
    # Project points to zero-level set using vector Adam
    optimized_points = []
    for p in initial_points:
        # Compute gradient at p
        grad = compute_gradient(udf_grid, p)
        if np.linalg.norm(grad) == 0:
            optimized_points.append(p)
            continue
        
        # Normalize gradient for direction constraint
        grad_norm = grad / np.linalg.norm(grad)
        
        # Optimize scalar step s along gradient direction
        # min_s u(p + s * grad_norm)
        s = vector_adam_optimize(udf_grid, p, grad_norm, lr, iterations)
        p_opt = p + s * grad_norm
        optimized_points.append(p_opt)
        
    # Identify Active Cubes based on occupancy
    active_cubes = set()
    for p_opt in optimized_points:
        # Determine which voxel contains p_opt
        voxel = get_voxel_index(p_opt, N)
        # The cube corresponding to this voxel is active
        # Note: In DCx, "cube" usually refers to the grid cell.
        # We mark the cube containing the optimized point as active.
        active_cubes.add(voxel)
        
    # Step 3: Supplementary Sampling
    # Find additional points where UDF < tau_1
    supplementary_points = []
    for i in range(N):
        for j in range(N):
            for k in range(N):
                if udf_grid[i, j, k] < tau_1:
                    p = np.array([i + 0.5, j + 0.5, k + 0.5])
                    # Check if already in initial points (approximate check)
                    if not is_in_set(initial_points, p):
                        supplementary_points.append(p)
                        
    for p in supplementary_points:
        grad = compute_gradient(udf_grid, p)
        if np.linalg.norm(grad) == 0:
            continue
        grad_norm = grad / np.linalg.norm(grad)
        s = vector_adam_optimize(udf_grid, p, grad_norm, lr, iterations)
        p_opt = p + s * grad_norm
        voxel = get_voxel_index(p_opt, N)
        active_cubes.add(voxel)
        
    # Step 4: Thinning
    # Identify expanded cubes (2x2x2 blocks)
    expanded_cubes = get_all_expanded_cubes(N)
    
    # Condition 1: Remove voxels violating Case 8 structural constraints
    # Iterate until no change
    changed = True
    while changed:
        changed = False
        for ec in expanded_cubes:
            if is_case_8(ec, active_cubes):
                for voxel in ec.voxels:
                    if not meets_case_8_validity(voxel, ec, active_cubes):
                        active_cubes.discard(voxel)
                        changed = True
                        
    # Condition 2: Remove redundant non-manifold voxels
    changed = True
    while changed:
        changed = False
        for ac in active_cubes:
            neighborhood = get_3x3x3_neighborhood(ac, N)
            for voxel in neighborhood:
                if voxel in active_cubes:
                    if is_redundant_non_manifold(voxel, active_cubes):
                        active_cubes.discard(voxel)
                        changed = True
                        
    # Step 5: Meshing via V2M Lookup Table
    vertices = []
    faces = []
    vertex_map = {} # Map voxel index to vertex index
    
    # Create vertices for active cubes
    for i, ac in enumerate(active_cubes):
        # Vertex position is the optimized point for this cube
        # Find the optimized point corresponding to this active cube
        # In implementation, store mapping from voxel to optimized point
        p_opt = get_optimized_point_for_voxel(ac)
        vertices.append(p_opt)
        vertex_map[ac] = i
        
    # Process each expanded cube to generate faces
    for ec in expanded_cubes:
        # Determine active/inactive status of 8 voxels in this expanded cube
        config = []
        for voxel in ec.voxels:
            if voxel in active_cubes:
                config.append(1)
            else:
                config.append(0)
                
        # Lookup table entry for this config
        face_indices = v2m_lookup_table[config]
        
        # Generate faces using the indices from the lookup table
        # The lookup table returns indices relative to the 8 voxels in the expanded cube
        for face in face_indices:
            # Map local voxel indices to global vertex indices
            global_indices = []
            for local_idx in face:
                voxel = ec.voxels[local_idx]
                if voxel in vertex_map:
                    global_indices.append(vertex_map[voxel])
                else:
                    # Should not happen if active cubes are correctly mapped
                    pass
                    
            if len(global_indices) == 3:
                faces.append(global_indices)
                
    # Step 6: Post-processing
    # Remove redundant non-manifold elements
    vertices, faces = post_process_mesh(vertices, faces)
    
    return vertices, faces

def vector_adam_optimize(udf_grid, p, grad_dir, lr, iterations):
    """
    Optimizes scalar step s along grad_dir to minimize u(p + s * grad_dir).
    Uses vector Adam optimizer.
    """
    # Initialize s, m, v
    s = 0.0
    m = 0.0
    v = 0.0
    beta1 = 0.9
    beta2 = 0.999
    eps = 1e-8
    
    for t in range(1, iterations + 1):
        # Compute UDF at current position
        pos = p + s * grad_dir
        u_val = udf_grid[get_voxel_index(pos, len(udf_grid))]
        
        # Compute gradient of UDF with respect to s
        # du/ds = grad_u . grad_dir
        grad_u = compute_gradient(udf_grid, pos)
        grad_s = np.dot(grad_u, grad_dir)
        
        # Update moments
        m = beta1 * m + (1 - beta1) * grad_s
        v = beta2 * v + (1 - beta2) * (grad_s ** 2)
        
        # Bias correction
        m_hat = m / (1 - beta1 ** t)
        v_hat = v / (1 - beta2 ** t)
        
        # Update s
        s = s - lr * m_hat / (np.sqrt(v_hat) + eps)
        
    return s

def compute_gradient(udf_grid, p):
    """
    Computes gradient at point p using central differences.
    """
    N = len(udf_grid)
    voxel = get_voxel_index(p, N)
    # Convert to continuous coordinates for interpolation if needed
    # For simplicity, assume p is close to voxel center
    # Use finite differences
    dx = 1.0 / N
    u = udf_grid[voxel]
    
    # Neighbors (handle boundaries)
    # ... implementation details for boundary checks ...
    
    grad_x = (udf_grid[voxel + (1,0,0)] - udf_grid[voxel - (1,0,0)]) / (2*dx)
    grad_y = (udf_grid[voxel + (0,1,0)] - udf_grid[voxel - (0,1,0)]) / (2*dx)
    grad_z = (udf_grid[voxel + (0,0,1)] - udf_grid[voxel - (0,0,1)]) / (2*dx)
    
    return np.array([grad_x, grad_y, grad_z])

def get_voxel_index(p, N):
    """
    Converts continuous point p to voxel index.
    """
    i = int(p[0] * N)
    j = int(p[1] * N)
    k = int(p[2] * N)
    # Clamp to grid bounds
    i = max(0, min(N-1, i))
    j = max(0, min(N-1, j))
    k = max(0, min(N-1, k))
    return (i, j, k)
```

## Subtopics

### [DCx Limitations and Constraints](9345-dcx-limitations-and-constraints.md)
DCx faces inherent topological and resolution constraints that prevent it from guaranteeing strictly manifold surfaces. It cannot natively support arbitrary non-manifold structures without post-processing, and its ability to resolve geometric details is strictly bound by grid resolution.

## Operating characteristics
- **Time Complexity:** The method involves optimization for each candidate point, which is the most expensive step. At resolution $N=256$, projection takes ~5984 ms, while V2M LUT lookup is fast (~107 ms) [S1]. The runtime scales super-linearly with resolution due to the optimization step (e.g., 14414 ms at 512, 61749 ms at 1024) [S1].
- **Memory Complexity:** Depends on the grid size $N^3$ for the UDF and the number of active cubes. The V2M lookup table is constant size ($2^8$ entries) [S1].
- **Strengths:** Produces single-layer, intersection-free meshes for both manifold and non-manifold inputs without supervision. More robust to noise and local minima than NDF (which uses SGD) and avoids double-layer entanglement of DCUDF [S1]. Conceptually simple and easy to implement [S1].
- **Limitations:** Cannot guarantee strictly manifold surfaces. High-curvature regions can cause topological artifacts due to ambiguity between high-curvature features and non-manifold geometry, though increasing resolution mitigates this [S1]. Performance is sensitive to UDF accuracy, especially in regions far from the zero-level set [S1].
- **When to use:** When extracting meshes from neural UDFs where non-manifold structures are present and a single-layer mesh is required.
- **When not to use:** When strict manifold topology is required without post-processing, or when high-resolution grids are computationally prohibitive.

## Background
DCx was proposed by Ying He et al. in 2026 as an extension of the standard Dual Contouring algorithm [Ju et al. 2002] to handle neural Unsigned Distance Functions (UDFs) [S1]. It addresses the challenges of extracting surfaces with non-manifold structures from UDFs, which lack sign information [S1]. The method was developed to overcome the limitations of existing methods like DCUDF, MeshUDF, NDC, and DMUDF, which either produce multi-layered meshes, fail to reconstruct non-manifold structures, or require supervision [S1].

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
