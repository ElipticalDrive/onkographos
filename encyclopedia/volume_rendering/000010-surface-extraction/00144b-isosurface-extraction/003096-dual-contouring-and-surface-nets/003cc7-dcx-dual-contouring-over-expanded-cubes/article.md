# DCx (Dual Contouring over Expanded Cubes)

Also called: Dual Contouring over Expanded Cubes

## Overview
DCx is an isosurface extraction method designed to reconstruct zero-level sets from neural Unsigned Distance Functions (UDFs), addressing the lack of sign information inherent in unsigned fields [S1]. Unlike standard Dual Contouring, which relies on sign changes to identify active cubes and connect vertices, DCx uses an optimization-based strategy to determine active cubes and operates on 2x2x2 voxel blocks (expanded cubes) with a voxel-to-mesh lookup table to handle sparse, irregular active cube distributions and non-manifold structures [S1]. It produces single-layer, intersection-free meshes for both manifold and non-manifold inputs without requiring supervision, avoiding the double-layer entanglement and topological errors common in prior UDF extraction methods [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A neural Unsigned Distance Function (UDF) $u(\mathbf{x})$ defined on a regular grid of resolution $N \times N \times N$.
*   **Output:** A single-layer triangle mesh representing the zero-level set $\{ \mathbf{x} \mid u(\mathbf{x}) = 0 \}$.

**Data Structures:**
*   **Grid:** Regular 3D grid with $N^3$ voxels. Each voxel stores the UDF value at its centroid.
*   **Expanded Cube:** A 2x2x2 block of voxels. Adjacent expanded cubes overlap, sharing 2x2x1 voxels (4 voxels in common).
*   **Dual Cube:** The 8 vertices of an expanded cube, treated as vertices in the dual space.
*   **V2M Lookup Table:** A mapping from the binary configuration of active/inactive voxels within an expanded cube to the mesh connectivity (faces/edges) on the dual cube vertices.
*   **Point Set:** A set of 3D points $\mathcal{P}$ approximating the zero-level set.

**Parameters:**
*   **Grid Resolution ($N$):** Typically 256, 512, 1024, or 2048 [S1].
*   **Thresholds:** $\tau_0 = 1.5/N$ and $\tau_1 = 0.5/N$ for $N=256$ [S1].
*   **Optimizer:** Vector Adam optimizer [S1].
*   **Learning Rate:** 0.01 [S1].
*   **Iterations:** 200 [S1].

**Algorithm Steps:**

1.  **Sampling and Optimization (Active Cube Identification):**
    *   Initialize a set of points $\mathcal{P}_{init}$ consisting of voxel centroids $\mathbf{c}_i$ where the UDF value $u(\mathbf{c}_i) \le \tau_0$ [S1].
    *   Refine $\mathcal{P}_{init}$ to $\mathcal{P}_{op}$ using direction-constrained optimization. For each point $\mathbf{p}_i \in \mathcal{P}_{init}$, solve the 1D line search:
        $$ \min_{s_i} u(\mathbf{p}_i + s_i \nabla u(\mathbf{p}_i)) $$
        where $\nabla u(\mathbf{p}_i)$ is the gradient of the UDF at $\mathbf{p}_i$, and $s_i$ is a scalar step size [S1].
    *   Use the Vector Adam optimizer with learning rate 0.01 for 200 iterations to find optimal $s_i$ [S1].
    *   Define the set of **Active Cubes** $\mathcal{C}_{active}$. A cube $C$ is active if its centroid $\mathbf{c}_C$ is in $\mathcal{P}_{op}$ (i.e., the projected point lies within the cube) [S1]. This naturally confines vertices to the local region, eliminating the need for clamping [S1].

2.  **Supplementary Sampling (Optional but Recommended):**
    *   To improve completeness in under-sampled regions, identify points $\mathbf{p}_j$ where $u(\mathbf{p}_j) < \tau_1$ but $\mathbf{p}_j \notin \mathcal{P}_{op}$ [S1].
    *   Apply the same direction-constrained optimization to these points and add them to $\mathcal{P}_{op}$ and $\mathcal{C}_{active}$ if they satisfy the active cube condition [S1].

3.  **Thinning (Noise and Artifact Removal):**
    *   Iterate over all expanded cubes classified as **Case 8** (a specific topological configuration in the V2M table).
    *   **Condition 1:** If a voxel in an expanded cube fails to meet the structural requirements for Case 8 validity, remove that voxel from the active set. Repeat until no voxels are removed [S1].
    *   **Condition 2:** For every active cube and its 3x3x3 neighborhood, remove a redundant voxel if all edges connected to it are boundary edges and at least one endpoint is on a non-manifold edge. Repeat until no such voxels are removed [S1].
    *   This step removes extraneous voxels that violate Case 8 constraints (preventing discontinuities) and eliminates superfluous non-manifold features [S1].

4.  **Meshing via V2M Lookup Table:**
    *   Iterate over all expanded cubes in the grid.
    *   For each expanded cube, determine the binary configuration of its 8 constituent voxels (active/inactive).
    *   Use the **V2M Lookup Table** to retrieve the corresponding mesh connectivity pattern for the dual cube vertices [S1].
    *   Generate triangles by connecting the active vertices (corresponding to active cubes) of the dual cube according to the retrieved pattern [S1].
    *   Aggregate case IDs of adjacent expanded cubes to ensure consistency, achieving an effective receptive field of 4x4x4 [S1].

5.  **Post-processing:**
    *   Remove residual redundant non-manifold elements and enhance mesh quality [S1].
    *   (Optional) Refine the extracted mesh using the optimization procedure from DCUDF to decrease Chamfer Distance, if the input UDF is not ground-truth [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def dcx_extract_mesh(udf_grid, N, tau0, tau1, lr, iterations):
    """
    Extracts a mesh from a neural UDF using DCx.
    
    Args:
        udf_grid: 3D numpy array of shape (N, N, N) containing UDF values.
        N: Grid resolution.
        tau0: Threshold for initial point selection.
        tau1: Threshold for supplementary sampling.
        lr: Learning rate for Vector Adam.
        iterations: Number of optimization iterations.
        
    Returns:
        vertices: Array of shape (V, 3) containing mesh vertices.
        faces: Array of shape (F, 3) containing mesh face indices.
    """
    
    # Step 1: Sampling and Optimization
    # Initialize point set with centroids where UDF <= tau0
    # Note: udf_grid values are assumed to be at voxel centers
    coords = np.argwhere(udf_grid <= tau0)
    points_init = coords.astype(float)
    
    # Compute gradients (finite difference or neural network gradient)
    # Assuming a function compute_gradient exists that returns gradient at each point
    gradients = compute_gradient(udf_grid, points_init)
    
    # Direction-constrained optimization using Vector Adam
    # Each point moves along its gradient direction: p_new = p + s * grad
    # We optimize scalar s for each point
    points_opt = vector_adam_optimize(points_init, gradients, udf_grid, lr, iterations)
    
    # Identify Active Cubes
    # A cube is active if its centroid is in points_opt
    # Centroids are integer coordinates (i, j, k)
    active_cube_set = set()
    for p in points_opt:
        # Round to nearest integer to find the cube index
        cube_idx = tuple(np.round(p).astype(int))
        if 0 <= cube_idx[0] < N and 0 <= cube_idx[1] < N and 0 <= cube_idx[2] < N:
            active_cube_set.add(cube_idx)
            
    # Step 2: Supplementary Sampling
    # Find points with UDF < tau1 not already in active set
    # This step is complex to implement efficiently without a neural network query
    # For simplicity, we assume we can query the UDF at any point
    supplementary_points = []
    # Scan grid for potential supplementary points
    for i in range(N):
        for j in range(N):
            for k in range(N):
                if (i, j, k) not in active_cube_set:
                    val = udf_grid[i, j, k]
                    if val < tau1:
                        # Add to supplementary list for optimization
                        supplementary_points.append([i, j, k])
                        
    if supplementary_points:
        supp_points = np.array(supplementary_points)
        supp_grads = compute_gradient(udf_grid, supp_points)
        supp_opt = vector_adam_optimize(supp_points, supp_grads, udf_grid, lr, iterations)
        
        for p in supp_opt:
            cube_idx = tuple(np.round(p).astype(int))
            if 0 <= cube_idx[0] < N and 0 <= cube_idx[1] < N and 0 <= cube_idx[2] < N:
                active_cube_set.add(cube_idx)
                
    # Step 3: Thinning
    # This requires iterating over expanded cubes and checking Case 8 conditions
    # We represent the active voxel set as a boolean grid
    active_voxel_grid = np.zeros((N, N, N), dtype=bool)
    for c in active_cube_set:
        active_voxel_grid[c] = True
        
    # Thinning Condition 1: Remove voxels violating Case 8 structural constraints
    # This is an iterative process. We define a helper to check Case 8 validity
    # For brevity, we assume a function thinning_condition_1 exists
    active_voxel_grid = thinning_condition_1(active_voxel_grid, active_cube_set)
    
    # Thinning Condition 2: Remove redundant voxels in 3x3x3 neighborhoods
    active_voxel_grid = thinning_condition_2(active_voxel_grid, active_cube_set)
    
    # Update active_cube_set based on thinned grid
    active_cube_set = set()
    active_cube_set = set(np.argwhere(active_voxel_grid))
    
    # Step 4: Meshing via V2M Lookup Table
    vertices = []
    faces = []
    vertex_map = {} # Map from cube_idx to vertex index
    
    # Create a map from cube index to a unique vertex index
    # Since each active cube produces one vertex, we can index them directly
    for idx, cube_idx in enumerate(active_cube_set):
        vertex_map[cube_idx] = idx
        # Vertex position is the optimized point for this cube
        # We need to retrieve the optimized point for this cube
        # In practice, we store the optimized points in a dict during optimization
        # Here we assume we can retrieve it
        p_opt = get_optimized_point(cube_idx, points_opt, supp_opt)
        vertices.append(p_opt)
        
    # Iterate over all possible expanded cubes (2x2x2 blocks)
    # An expanded cube is defined by its top-left corner (i, j, k)
    # where 0 <= i, j, k <= N-2
    for i in range(N - 1):
        for j in range(N - 1):
            for k in range(N - 1):
                # Get the 8 voxels in this expanded cube
                voxel_indices = []
                for di in range(2):
                    for dj in range(2):
                        for dk in range(2):
                            voxel_indices.append((i + di, j + dj, k + dk))
                            
                # Determine active status of each voxel
                active_mask = [active_voxel_grid[v] for v in voxel_indices]
                
                # Convert mask to integer case ID (0-255)
                case_id = 0
                for bit, is_active in enumerate(active_mask):
                    if is_active:
                        case_id |= (1 << bit)
                        
                # Lookup mesh connectivity for this case
                # The V2M table returns a list of faces (indices into the 8 vertices of the dual cube)
                faces_local = v2m_lookup_table[case_id]
                
                # Map local dual cube vertices to global vertex indices
                # The 8 vertices of the dual cube correspond to the 8 cubes in the expanded cube
                # Note: The expanded cube spans 2x2x2 voxels, which are 8 cubes
                # We need to map each voxel index to its corresponding cube index (which is the same)
                # And then to the global vertex index
                global_faces = []
                for face in faces_local:
                    global_face = []
                    for local_v_idx in face:
                        voxel_idx = voxel_indices[local_v_idx]
                        if voxel_idx in active_cube_set:
                            global_face.append(vertex_map[voxel_idx])
                        else:
                            # This should not happen if the V2M table is consistent with active mask
                            # But if it does, we skip this face
                            global_face = None
                            break
                    if global_face is not None:
                        global_faces.append(global_face)
                        
                faces.extend(global_faces)
                
    return np.array(vertices), np.array(faces)

def vector_adam_optimize(points, gradients, udf_grid, lr, iterations):
    """
    Performs direction-constrained optimization using Vector Adam.
    Each point moves along its gradient direction.
    """
    # Initialize scalar steps s_i = 0 for all points
    s = np.zeros(len(points))
    
    # Adam state variables
    m_s = np.zeros_like(s)
    v_s = np.zeros_like(s)
    t = 0
    
    for _ in range(iterations):
        t += 1
        
        # Compute current points
        current_points = points + s[:, np.newaxis] * gradients
        
        # Compute UDF values at current points
        # This requires querying the UDF at arbitrary 3D points
        udf_values = query_udf(udf_grid, current_points)
        
        # Compute gradients of UDF with respect to s_i
        # d/ds u(p + s * grad) = grad_u . grad_p
        # We assume compute_gradient returns the spatial gradient
        # The directional derivative is the dot product of spatial gradient and search direction (spatial gradient)
        # So it's ||grad_u||^2
        # However, for Adam, we need the gradient of the loss w.r.t s
        # Loss = u(p + s * grad)
        # dLoss/ds = grad_u(p + s*grad) . grad
        # We approximate this by evaluating grad_u at the current point
        current_grads = compute_gradient(udf_grid, current_points)
        grad_s = np.sum(current_grads * gradients, axis=1)
        
        # Update Adam state
        m_s = 0.9 * m_s + 0.1 * grad_s
        v_s = 0.999 * v_s + 0.001 * grad_s**2
        
        m_hat = m_s / (1 - 0.9**t)
        v_hat = v_s / (1 - 0.999**t)
        
        s -= lr * m_hat / (np.sqrt(v_hat) + 1e-8)
        
    return points + s[:, np.newaxis] * gradients

def query_udf(udf_grid, points):
    """
    Queries the UDF grid at arbitrary 3D points using trilinear interpolation.
    """
    # Clip points to grid bounds
    points = np.clip(points, 0, udf_grid.shape[0] - 1)
    
    # Trilinear interpolation
    # This is a standard operation, omitted for brevity
    # Return interpolated values
    return np.zeros(len(points)) # Placeholder

def compute_gradient(udf_grid, points):
    """
    Computes the gradient of the UDF at given points.
    """
    # This is a standard operation, omitted for brevity
    # Return gradients
    return np.zeros((len(points), 3)) # Placeholder

def thinning_condition_1(active_voxel_grid, active_cube_set):
    """
    Removes voxels that violate Case 8 structural constraints.
    """
    # Implementation detail: Check each expanded cube for Case 8 validity
    # If a voxel is part of an expanded cube that is Case 8 but the voxel itself
    # does not meet the structural condition, remove it.
    # This is an iterative process.
    changed = True
    while changed:
        changed = False
        # Iterate over all expanded cubes
        for i in range(active_voxel_grid.shape[0] - 1):
            for j in range(active_voxel_grid.shape[1] - 1):
                for k in range(active_voxel_grid.shape[2] - 1):
                    # Check if this expanded cube is Case 8
                    # ...
                    # If so, check each voxel
                    # If a voxel violates condition, remove it
                    pass
    return active_voxel_grid

def thinning_condition_2(active_voxel_grid, active_cube_set):
    """
    Removes redundant voxels where all connected edges are boundary edges
    and at least one endpoint is on a non-manifold edge.
    """
    # Implementation detail: Iterate over active voxels and their 3x3x3 neighborhood
    # Check the condition
    # If met, remove the voxel
    changed = True
    while changed:
        changed = False
        # ...
        pass
    return active_voxel_grid

# V2M Lookup Table
# This is a predefined table mapping case_id (0-255) to a list of faces
# Each face is a list of 3 indices (0-7) corresponding to the 8 voxels in the expanded cube
v2m_lookup_table = {
    # Example: Case 0 (no active voxels) -> no faces
    0: [],
    # Case 1 (only voxel 0 active) -> no faces (single vertex)
    1: [],
    # ...
    # Case 8 (specific configuration) -> list of faces
    8: [[0, 1, 2], [0, 2, 3]], # Example
    # ...
    # Full table would have 256 entries
}

def get_optimized_point(cube_idx, points_opt, supp_opt):
    """
    Retrieves the optimized point for a given cube index.
    """
    # This requires storing the mapping from cube index to optimized point
    # Omitted for brevity
    return np.array([0.0, 0.0, 0.0])
```

## Subtopics

### [DCx Limitations and Constraints](9345-dcx-limitations-and-constraints.md)
DCx faces inherent topological and resolution constraints that prevent it from guaranteeing strictly manifold surfaces. Unlike SDFs, it cannot natively support arbitrary non-manifold structures without post-processing, and its ability to resolve geometric details is strictly bound by grid resolution.

## Operating characteristics
*   **Time Complexity:** Dominated by the projection step (optimization) and supplementary sampling. At resolution $N=256$, projection takes ~5984 ms, supplementary sampling ~1566 ms, and thinning ~519 ms [S1]. The V2M lookup table is fast (~107 ms) [S1].
*   **Memory Complexity:** Proportional to the grid resolution $N^3$ for storing the UDF and active voxel grid. The V2M lookup table is constant size (256 entries).
*   **Performance:** DCx produces meshes with higher accuracy and robustness than existing methods (DCUDF, NDC, DMUDF) on both manifold and non-manifold inputs [S1]. It avoids the double-layer entanglement of DCUDF and the multi-layer artifacts of NDC/DMUDF [S1].
*   **Strengths:** Handles non-manifold structures, produces single-layer meshes, robust to noise due to optimization, simple to implement.
*   **Limitations:** Resolution-dependent detail, potential topological artifacts in high-curvature regions (mitigated by higher resolution), relies on accurate UDF gradients.
*   **When to use:** When extracting meshes from neural UDFs, especially for non-manifold geometries.
*   **When not to use:** When strict manifold guarantee is required without post-processing, or when extremely high-frequency details are needed beyond the grid resolution.

## Background
DCx was proposed as an extension of the standard Dual Contouring algorithm [Ju et al. 2002] to address the challenges of extracting surfaces from neural Unsigned Distance Functions (UDFs) [S1]. Standard Dual Contouring relies on sign changes in SDFs, which are absent in UDFs. DCx was introduced to enable the reconstruction of both manifold and non-manifold surfaces from UDFs in a single-layer mesh representation, avoiding the topological errors and double-layer entanglement of prior methods like DCUDF [S1]. The method was presented in a 2026 ACM paper [S1].

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
