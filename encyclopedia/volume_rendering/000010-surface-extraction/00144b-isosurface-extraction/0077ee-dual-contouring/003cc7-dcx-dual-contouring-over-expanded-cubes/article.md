# DCx (Dual Contouring over Expanded Cubes)

Also called: Dual Contouring over Expanded Cubes

## Overview
DCx is an isosurface extraction method designed to reconstruct zero-level sets from neural Unsigned Distance Functions (UDFs), addressing the lack of sign information inherent in unsigned fields [S1]. Unlike standard Dual Contouring which relies on sign changes to identify active cubes and connect vertices, DCx uses an optimization-based strategy for active cube detection and operates on 2x2x2 voxel blocks (expanded cubes) to robustly handle sparse, irregular active cube distributions and non-manifold structures [S1]. It produces single-layer, intersection-free meshes for both manifold and non-manifold inputs without requiring supervision, avoiding the double-layer entanglement and topological errors common in prior UDF extraction methods [S1].

## How it works
**Inputs:**
- A neural Unsigned Distance Function (UDF) $u(\mathbf{x})$ defined on a 3D grid [S1].
- Grid resolution $N$ (e.g., 256) [S1].
- Thresholds $\tau_0$ and $\tau_1$ for active cube detection and supplementary sampling [S1].

**Outputs:**
- A single-layer triangle mesh representing the zero-level set of the UDF [S1].

**Data Structures:**
- **Grid:** A 3D array of size $N \times N \times N$ storing UDF values [S1].
- **Expanded Cube:** A 2x2x2 block of voxels [S1]. Adjacent expanded cubes overlap, sharing 2x2x1 voxels (4 voxels in common) [S1].
- **V2M Lookup Table:** A mapping from the binary configuration of 8 voxels (active/inactive) within an expanded cube to the corresponding mesh connectivity (faces/edges) on the dual cube vertices [S1].
- **Active Cube Set:** A set of cube indices identified as active based on the optimization step [S1].
- **Point Set:** A set of 3D points $\mathbf{p}$ representing projected centroids [S1].

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
    - Let $\mathbf{p}_i^{opt} = \mathbf{p}_i + s_i \nabla u(\mathbf{p}_i)$ be the optimized point [S1].
    - Identify **Active Cubes**: A cube is marked as active if its centroid's optimized point $\mathbf{p}_i^{opt}$ lies within the cube boundaries [S1]. This defines active cubes based on occupancy, naturally confining vertices and eliminating the need for corrective clamping [S1].

2.  **Supplementary Sampling:**
    - To improve completeness in under-sampled regions, perform supplementary sampling [S1]. Select additional voxel centroids where $u(\mathbf{c}_j) < \tau_1$ [S1].
    - Apply the same direction-constrained optimization (vector Adam) to these points [S1].
    - Mark cubes containing these optimized points as active if they satisfy the occupancy condition [S1].

3.  **Thinning (Noise and Artifact Removal):**
    - Iterate over all expanded cubes classified as Case 8 (a specific topological configuration in the V2M table) [S1].
    - **Condition 1:** If a voxel in a Case 8 configuration fails to meet structural requirements for Case 8 validity, delete that voxel [S1]. Repeat until no further voxels are removed [S1].
    - **Condition 2:** For every active cube and its 3x3x3 neighborhood, remove a redundant voxel if all edges connected to the active voxel are boundary edges and at least one endpoint is on a non-manifold edge [S1]. Repeat until no such voxels are removed [S1].
    - This step removes noise, eliminates redundant non-manifold artifacts, and mitigates holes caused by local configurations violating Case 8 rules [S1].

4.  **Meshing via V2M Lookup Table:**
    - Examine all expanded cubes spanning 2x2x2 voxels [S1].
    - For each expanded cube, determine the binary configuration of its 8 voxels (active or inactive) [S1].
    - Use the V2M Lookup Table to retrieve the predefined mesh connectivity pattern for that configuration [S1].
    - Generate faces by connecting the active vertices (corresponding to the active cubes) of the dual cube (treating the 8 cubes as 8 vertices) according to the lookup table [S1].
    - Aggregate the case IDs of adjacent expanded cubes to achieve a polygonization criterion with an effective receptive field of 4x4x4, ensuring consistency across overlapping regions [S1].

5.  **Post-processing:**
    - Apply post-processing to efficiently remove residual redundant non-manifold elements and enhance overall mesh quality [S1].
    - For learned neural UDFs, refine the extracted mesh using the optimization procedure from DCUDF to decrease Chamfer Distance; this step is skipped for ground-truth UDFs [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dcx_extract_mesh(udf_grid, N, tau_0, tau_1, lr=0.01, iterations=200):
    """
    Extracts a mesh from a neural UDF using DCx.
    
    Args:
        udf_grid: 3D list of UDF values, shape N x N x N.
        N: Grid resolution.
        tau_0: Threshold for initial active cube detection.
        tau_1: Threshold for supplementary sampling.
        lr: Learning rate for Adam optimizer.
        iterations: Number of optimization iterations.
        
    Returns:
        vertices: List of vertex positions (tuples).
        faces: List of face indices (tuples of 3 ints).
    """
    
    # Step 1: Initial Sampling
    initial_points = []
    for i in range(N):
        for j in range(N):
            for k in range(N):
                if udf_grid[i][j][k] <= tau_0:
                    p = (i + 0.5, j + 0.5, k + 0.5)
                    initial_points.append(p)
    
    # Step 2: Direction-Constrained Optimization
    optimized_points = []
    point_to_voxel = {}
    for p in initial_points:
        grad = compute_gradient(udf_grid, p, N)
        if norm(grad) == 0:
            optimized_points.append(p)
            continue
        grad_dir = normalize(grad)
        s = vector_adam_optimize(udf_grid, p, grad_dir, lr, iterations, N)
        p_opt = tuple(p[d] + s * grad_dir[d] for d in range(3))
        optimized_points.append(p_opt)
        voxel = get_voxel_index(p_opt, N)
        point_to_voxel[voxel] = p_opt
        
    # Identify Active Cubes based on occupancy
    active_cubes = set(point_to_voxel.keys())
        
    # Step 3: Supplementary Sampling
    supplementary_points = []
    for i in range(N):
        for j in range(N):
            for k in range(N):
                if udf_grid[i][j][k] < tau_1:
                    p = (i + 0.5, j + 0.5, k + 0.5)
                    if p not in initial_points:
                        supplementary_points.append(p)
                        
    for p in supplementary_points:
        grad = compute_gradient(udf_grid, p, N)
        if norm(grad) == 0:
            continue
        grad_dir = normalize(grad)
        s = vector_adam_optimize(udf_grid, p, grad_dir, lr, iterations, N)
        p_opt = tuple(p[d] + s * grad_dir[d] for d in range(3))
        voxel = get_voxel_index(p_opt, N)
        active_cubes.add(voxel)
        point_to_voxel[voxel] = p_opt
        
    # Step 4: Thinning
    # Condition 1: Remove voxels violating Case 8 structural constraints
    changed = True
    while changed:
        changed = False
        for i in range(N - 1):
            for j in range(N - 1):
                for k in range(N - 1):
                    ec_voxels = get_expanded_cube_voxels(i, j, k, N)
                    if is_case_8(ec_voxels, active_cubes):
                        for v in ec_voxels:
                            if v in active_cubes and not meets_case_8_validity(v, ec_voxels, active_cubes):
                                active_cubes.discard(v)
                                changed = True
                                
    # Condition 2: Remove redundant non-manifold voxels
    changed = True
    while changed:
        changed = False
        for v in list(active_cubes):
            neighborhood = get_3x3x3_neighborhood(v, N)
            if is_redundant_non_manifold(v, neighborhood, active_cubes):
                active_cubes.discard(v)
                changed = True
                
    # Step 5: Meshing via V2M Lookup Table
    vertices = []
    faces = []
    vertex_map = {}
    
    # Create vertices for active cubes
    for idx, v in enumerate(active_cubes):
        vertices.append(point_to_voxel[v])
        vertex_map[v] = idx
        
    # Process each expanded cube to generate faces
    for i in range(N - 1):
        for j in range(N - 1):
            for k in range(N - 1):
                ec_voxels = get_expanded_cube_voxels(i, j, k, N)
                config = tuple(1 if v in active_cubes else 0 for v in ec_voxels)
                face_indices = V2M_LOOKUP_TABLE[config]
                
                for face in face_indices:
                    global_indices = []
                    for local_idx in face:
                        v = ec_voxels[local_idx]
                        if v in vertex_map:
                            global_indices.append(vertex_map[v])
                    if len(global_indices) == 3:
                        faces.append(tuple(global_indices))
                        
    # Step 6: Post-processing
    vertices, faces = post_process_mesh(vertices, faces)
    
    return vertices, faces

def vector_adam_optimize(udf_grid, p, grad_dir, lr, iterations, N):
    """
    Optimizes scalar step s along grad_dir to minimize u(p + s * grad_dir).
    Uses vector Adam optimizer.
    """
    s = 0.0
    m = 0.0
    v = 0.0
    beta1 = 0.9
    beta2 = 0.999
    eps = 1e-8
    
    for t in range(1, iterations + 1):
        pos = tuple(p[d] + s * grad_dir[d] for d in range(3))
        u_val = udf_grid[get_voxel_index(pos, N)[0]][get_voxel_index(pos, N)[1]][get_voxel_index(pos, N)[2]]
        
        grad_u = compute_gradient(udf_grid, pos, N)
        grad_s = dot(grad_u, grad_dir)
        
        m = beta1 * m + (1 - beta1) * grad_s
        v = beta2 * v + (1 - beta2) * (grad_s ** 2)
        
        m_hat = m / (1 - beta1 ** t)
        v_hat = v / (1 - beta2 ** t)
        
        s = s - lr * m_hat / (sqrt(v_hat) + eps)
        
    return s

def compute_gradient(udf_grid, p, N):
    """
    Computes gradient at point p using central differences.
    """
    voxel = get_voxel_index(p, N)
    i, j, k = voxel
    dx = 1.0 / N
    
    def get_val(ii, jj, kk):
        ii = max(0, min(N - 1, ii))
        jj = max(0, min(N - 1, jj))
        kk = max(0, min(N - 1, kk))
        return udf_grid[ii][jj][kk]
        
    u = get_val(i, j, k)
    grad_x = (get_val(i + 1, j, k) - get_val(i - 1, j, k)) / (2 * dx)
    grad_y = (get_val(i, j + 1, k) - get_val(i, j - 1, k)) / (2 * dx)
    grad_z = (get_val(i, j, k + 1) - get_val(i, j, k - 1)) / (2 * dx)
    
    return (grad_x, grad_y, grad_z)

def get_voxel_index(p, N):
    """
    Converts continuous point p to voxel index.
    """
    i = int(p[0] * N)
    j = int(p[1] * N)
    k = int(p[2] * N)
    i = max(0, min(N - 1, i))
    j = max(0, min(N - 1, j))
    k = max(0, min(N - 1, k))
    return (i, j, k)

def get_expanded_cube_voxels(i, j, k, N):
    """Returns the 8 voxel indices of the 2x2x2 block starting at (i,j,k)."""
    voxels = []
    for di in range(2):
        for dj in range(2):
            for dk in range(2):
                voxels.append((i + di, j + dj, k + dk))
    return voxels

def get_3x3x3_neighborhood(v, N):
    """Returns the 27 voxel indices in the 3x3x3 neighborhood of v."""
    i, j, k = v
    neighborhood = []
    for di in range(-1, 2):
        for dj in range(-1, 2):
            for dk in range(-1, 2):
                ni, nj, nk = i + di, j + dj, k + dk
                if 0 <= ni < N and 0 <= nj < N and 0 <= nk < N:
                    neighborhood.append((ni, nj, nk))
    return neighborhood

def is_case_8(ec_voxels, active_cubes):
    """Placeholder: Returns True if the expanded cube matches Case 8 topology.
    The exact topological classification logic is implementation-specific."""
    return True

def meets_case_8_validity(v, ec_voxels, active_cubes):
    """Placeholder: Checks if voxel v satisfies Case 8 structural constraints.
    The exact validity conditions are implementation-specific."""
    return True

def is_redundant_non_manifold(v, neighborhood, active_cubes):
    """Placeholder: Checks Condition 2 for thinning.
    Returns True if all edges connected to v are boundary edges and at least one endpoint is on a non-manifold edge."""
    return False

def post_process_mesh(vertices, faces):
    """Placeholder: Removes residual redundant non-manifold elements and enhances mesh quality."""
    return vertices, faces

# V2M Lookup Table: Maps 8-bit voxel configurations to face indices.
# Contains 2^8 entries. Each entry is a list of 3-tuples representing triangle indices
# relative to the 8 voxels in the expanded cube.
V2M_LOOKUP_TABLE = {config: [] for config in range(256)}
```

## Subtopics

### [DCx Limitations and Constraints](9345-dcx-limitations-and-constraints.md)
DCx faces inherent topological and resolution constraints that prevent it from guaranteeing strictly manifold surfaces. It cannot natively support arbitrary non-manifold structures without post-processing, and its ability to resolve geometric details is strictly bound by grid resolution.

## Operating characteristics
- **Time Complexity:** The method involves optimization for each candidate point, which is the most expensive step [S1]. At resolution $N=256$, projection takes ~5984 ms, while V2M LUT lookup is fast (~107 ms) [S1]. Measured runtimes at other resolutions are: 14414 ms projection at 512, 61749 ms at 1024, and 474778 ms at 2048 [S1].
- **Memory Complexity:** Depends on the grid size $N^3$ for the UDF and the number of active cubes [S1]. The V2M lookup table is constant size ($2^8$ entries) [S1].
- **Strengths:** Produces single-layer, intersection-free meshes for both manifold and non-manifold inputs without supervision [S1]. More robust to noise and local minima than NDF (which uses SGD) and avoids double-layer entanglement of DCUDF [S1]. Conceptually simple and easy to implement [S1].
- **Limitations:** Cannot guarantee strictly manifold surfaces [S1]. High-curvature regions can cause topological artifacts due to ambiguity between high-curvature features and non-manifold geometry; increasing the grid resolution significantly mitigates these errors without prohibitive overhead [S1]. Performance is sensitive to UDF accuracy, especially in regions far from the zero-level set [S1].
- **When to use:** When extracting meshes from neural UDFs where non-manifold structures are present and a single-layer mesh is required [S1].
- **When not to use:** When strict manifold topology is required without post-processing, or when high-resolution grids are computationally prohibitive [S1].

## Background
DCx was proposed by Ying He et al. in 2026 as an extension of the standard Dual Contouring algorithm [Ju et al. 2002] to handle neural Unsigned Distance Functions (UDFs) [S1]. It addresses the challenges of extracting surfaces with non-manifold structures from UDFs, which lack sign information [S1]. The method was developed to overcome the limitations of existing methods like DCUDF, MeshUDF, NDC, and DMUDF, which either produce multi-layered meshes, fail to reconstruct non-manifold structures, or require supervision [S1].

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
