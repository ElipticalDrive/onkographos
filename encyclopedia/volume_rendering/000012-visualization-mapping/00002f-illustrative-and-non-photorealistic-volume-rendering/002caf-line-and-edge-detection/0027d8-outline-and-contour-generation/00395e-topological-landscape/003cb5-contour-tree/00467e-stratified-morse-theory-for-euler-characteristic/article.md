# Stratified Morse Theory for Euler Characteristic

Also called: MMS-SMT, Method of Manufactured Solutions (MMS) using Stratified Morse Theory, SMT Verification, Stratified Morse Theory (SMT) Verification

## Overview
Stratified Morse Theory (SMT) for Euler Characteristic is a method to compute the Euler characteristic $\chi$ of isosurfaces defined by piecewise trilinear interpolants on regular grids, specifically handling the non-smoothness at cell boundaries [S1, unofficial]. It serves two primary purposes: verifying the topological correctness of isosurface extraction algorithms via a Manufactured Solutions pipeline, and estimating noise levels in direct volume rendering transfer function design [S1, S2, unofficial]. Unlike smooth Morse theory, which fails on discontinuous derivatives, SMT stratifies the grid into vertices, edges, faces, and cells to apply local topological analysis [S1, unofficial]. It is distinct from the [Contour Tree](9312-contour-tree.md) in that it provides a scalar topological invariant ($\chi$) for verification or noise estimation rather than a hierarchical graph of critical points, and differs from [Reeb Graph](9315-reeb-graph.md) by focusing on the Euler characteristic specifically rather than component merging/splitting paths [S1, S2, unofficial].

## How it works
The method operates in two distinct contexts: **Verification** (computing $\chi$ from critical points of a trilinear field) and **Noise Estimation** (computing Total Euler Characteristic from an image/volume).

### Inputs and Outputs
*   **Verification Input:** A rectilinear grid $G$ of size $n \times n \times n$ with scalar values sampled in $[-1, 1]$ [S1, unofficial].
*   **Verification Output:** The Euler characteristic $\chi$ of the isosurface at a specific isovalue (typically 0) [S1, unofficial].
*   **Noise Input:** An integer-valued image (2D) or volume (3D) (e.g., 8-bit or 16-bit) [S2, unofficial].
*   **Noise Output:** The Total Euler Characteristic (TEC), a scalar noise measure [S2, unofficial].

### Data Structures
1.  **Stratified Grid:** The domain is partitioned into four strata:
    *   $V$: Vertex set (0-dimensional).
    *   $E$: Edge set (1-dimensional interiors).
    *   $F$: Face set (2-dimensional interiors).
    *   $C$: Cell set (3-dimensional interiors) [S1, unofficial].
2.  **Histogram (Noise Context):** An array $H[T]$ indexed by intensity value $T$, with bin width 1. Size depends on data type (e.g., 256 for 8-bit) [S2, unofficial].

### Algorithm: Verification (MMS-SMT)
This pipeline verifies an isosurfacing algorithm by comparing the $\chi$ computed from the mesh against the $\chi$ computed from SMT critical points [S1, unofficial].

1.  **Generate Grid:** Let $G$ be an $n \times n \times n$ rectilinear grid.
2.  **Loop Tests:** For $i$ from 1 to `#tests`:
    3.  **Sample:** Randomly sample scalar values for $G$ from $[-1, 1]$ [S1, unofficial].
    4.  **Compute Critical Points (CPs):**
        *   Identify points with zero gradient on each stratum (Edge, Face, Cell) [S1, unofficial].
        *   Include all points in the Vertex set as critical points [S1, unofficial].
    5.  **Check Degeneracy:** If any critical point $p$ is degenerate or is an internal saddle close to edges/faces, restart from step 3 [S1, unofficial].
    6.  **Extract Mesh:** Extract isosurface $K$ from $G$ using the algorithm under verification [S1, unofficial].
    7.  **Compute $\chi_v$ (SMT):** Calculate $\chi$ from critical points using the stratified formulas below [S1, unofficial].
    8.  **Compute $\chi_k$ (Mesh):** Calculate $\chi$ from mesh $K$ using $\chi(M) = V(M) - E(M) + F(M)$, where $V, E, F$ are counts of vertices, edges, and triangles [S1, unofficial].
    9.  **Compare:** If $\chi_v \neq \chi_k$, verification fails [S1, unofficial].

### Algorithm: Noise Estimation (TEC)
This method computes the evolution of $\chi$ for all isovalues simultaneously using a raster scan [S2, unofficial].

1.  **Initialize:** Create histogram $H[T]$ with bin width 1 for all possible intensity values $I$ [S2, unofficial].
2.  **Raster Scan:** Perform a single pass over the image/volume using a sliding window:
    *   **2D:** $2 \times 2$ neighborhood.
    *   **3D:** $2 \times 2 \times 2$ neighborhood [S2, unofficial].
3.  **Update Histogram:** For each neighborhood, update $H$ based on the topology of the super level set as the isovalue decreases from maximum to minimum:
    *   **Vertices:** For each vertex in the neighborhood, increment $H[I(x,y)]$ (2D) or $H[I(x,y,z)]$ (3D). This corresponds to the creation of a connected component (island) [S2, unofficial].
    *   **Edges (2D) / Faces (3D):**
        *   In 2D, for each edge, decrement $H[\min(I_{neighbor1}, I_{neighbor2})]$. This corresponds to the creation of a hole (or merging of components) [S2, unofficial].
        *   In 3D, for each face, increment $H[\min(I_{neighbors})]$. This corresponds to the creation of a tunnel (Betti-1 feature) [S2, unofficial].
    *   **Cells (3D):** For each voxel (cell), decrement $H[\min(I_{voxel})]$. This corresponds to the closure of a tunnel [S2, unofficial].
4.  **Accumulate:** After the scan, accumulate $H$ from the highest bin downwards. Each bin $H[T]$ now contains the cumulative Euler characteristic $\chi(T)$ for the super level set at isovalue $T$ [S2, unofficial].
5.  **Compute TEC:** Calculate $TEC = \sum_T |\chi(T)|$ [S2, unofficial].

### Critical Point Analysis (Verification Context)
For a critical point $p$, the change in Euler characteristic is $\Delta\chi(p) = \chi(L^+(p)) - \chi(L^-(p))$ [S1, unofficial]. The links $L^-$ and $L^+$ are defined by tangential ($T$) and normal ($N$) submanifolds [S1, unofficial].

**Recursive Formula:**
$$ \chi(L^-) = \chi(N_{L^-}) + \chi(T_{L^-}) - \chi(N_{L^-})\chi(T_{L^-}) $$
This is applied recursively until 1-disks are reached [S1, unofficial].

**Specific Cases:**

1.  **Vertex Critical Points:**
    Let $nl_1, nl_2, nl_3$ be the number of adjacent vertices with scalar values less than $p$ in the $x, y, z$ directions respectively [S1, unofficial].
    $$ \chi(L^-(p)) = nl_1 + nl_2 + nl_3 - nl_1(nl_2 + nl_3) $$
    $$ \chi(L^+(p)) \text{ is computed similarly using neighbors with values greater than } p $$ [S1, unofficial].

2.  **Face Critical Points (Saddles):**
    Let $r$ be the face containing $p$. Let $r^\perp$ be the line segment orthogonal to $r$ through $p$.
    Let $nl$ be the number of ends of $r^\perp$ with scalar value less than $p$.
    Let $nu$ be the number of ends of $r^\perp$ with scalar value higher than $p$.
    $$ \chi(L^-(p)) = nl + 2 - 2nl $$
    $$ \chi(L^+(p)) = nu + 2 - 2nu $$ [S1, unofficial].

3.  **Edge/Critical Points in Cell Interiors:**
    *   Edge critical points are found by solving linear equations derived from the trilinear interpolant. Solutions outside the unit cube are ignored [S1, unofficial].
    *   Cell interior critical points are handled by smooth Morse theory (Hessian eigenvalues) as the function is smooth within the cell [S1, unofficial].

### Pseudocode: Verification (MMS-SMT)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def verify_isosurface(grid, isosurfacing_func, num_tests):
    """
    Verifies an isosurfacing algorithm using Stratified Morse Theory.
    grid: n x n x n rectilinear grid with scalar values in [-1, 1]
    isosurfacing_func: function that extracts a mesh from grid
    num_tests: number of random tests to perform
    """
    for i in range(num_tests):
        # 1. Randomly sample grid
        grid = random_sample_grid(grid)
        
        # 2. Compute Critical Points
        critical_points = compute_stratified_critical_points(grid)
        
        # 3. Check for degeneracy
        if has_degenerate_critical_point(critical_points):
            continue # Restart test
        
        # 4. Extract Mesh
        mesh = isosurfacing_func(grid, isovalue=0)
        
        # 5. Compute Chi from Mesh
        chi_mesh = compute_mesh_euler_characteristic(mesh)
        
        # 6. Compute Chi from SMT
        chi_smt = 0
        for p in critical_points:
            delta_chi = compute_critical_point_delta_chi(p, grid)
            chi_smt += delta_chi
            
        # 7. Compare
        if chi_smt != chi_mesh:
            return False, "Verification failed: Chi mismatch"
            
    return True, "Verification passed"

def compute_critical_point_delta_chi(p, grid):
    """
    Computes Delta Chi for a single critical point p.
    """
    stratum = get_stratum(p)
    
    if stratum == 'vertex':
        nl1 = count_adjacent_vertices_less_than(p, axis=0)
        nl2 = count_adjacent_vertices_less_than(p, axis=1)
        nl3 = count_adjacent_vertices_less_than(p, axis=2)
        chi_L_minus = nl1 + nl2 + nl3 - nl1 * (nl2 + nl3)
        
        nu1 = count_adjacent_vertices_greater_than(p, axis=0)
        nu2 = count_adjacent_vertices_greater_than(p, axis=1)
        nu3 = count_adjacent_vertices_greater_than(p, axis=2)
        chi_L_plus = nu1 + nu2 + nu3 - nu1 * (nu2 + nu3)
        
    elif stratum == 'face':
        # p lies in face r
        nl = count_ends_orthogonal_segment_less_than(p, grid)
        nu = count_ends_orthogonal_segment_greater_than(p, grid)
        
        chi_L_minus = nl + 2 - 2 * nl
        chi_L_plus = nu + 2 - 2 * nu
        
    elif stratum == 'edge':
        # Solve for critical point on edge, compute links recursively
        # Simplified: assume smooth theory applies if not degenerate
        # Or use recursive stratified formula for 1D stratum
        chi_L_minus = compute_recursive_link(p, 'minus', grid)
        chi_L_plus = compute_recursive_link(p, 'plus', grid)
        
    elif stratum == 'cell':
        # Smooth Morse Theory
        hessian = compute_hessian(p, grid)
        eigenvalues = compute_eigenvalues(hessian)
        chi_L_minus = compute_smooth_chi_minus(eigenvalues)
        chi_L_plus = compute_smooth_chi_plus(eigenvalues)
        
    else:
        return 0
        
    return chi_L_plus - chi_L_minus

def compute_recursive_link(p, direction, grid):
    """
    Recursively computes Chi(L) using the formula:
    Chi(L) = Chi(N) + Chi(T) - Chi(N)*Chi(T)
    """
    # Determine Tangential (T) and Normal (N) submanifolds
    T_submanifold = get_tangential_submanifold(p, grid)
    N_submanifold = get_normal_submanifold(p, grid)
    
    # Compute Chi for T and N
    # Base case: if T or N are 1-disks, Chi is cardinality of boundary
    if is_1_disk(T_submanifold):
        chi_T = cardinality(boundary(T_submanifold, direction))
    else:
        chi_T = compute_recursive_link(T_submanifold, direction, grid)
        
    if is_1_disk(N_submanifold):
        chi_N = cardinality(boundary(N_submanifold, direction))
    else:
        chi_N = compute_recursive_link(N_submanifold, direction, grid)
        
    return chi_N + chi_T - chi_N * chi_T
```

### Pseudocode: Noise Estimation (TEC)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_total_euler_characteristic(image):
    """
    Computes Total Euler Characteristic (TEC) for noise estimation.
    image: 2D or 3D integer-valued array
    """
    # Determine dimensions
    is_3d = image.ndim == 3
    
    # Initialize histogram
    max_val = image.max()
    min_val = image.min()
    H = np.zeros(max_val - min_val + 1)
    
    # Raster scan with sliding window
    if is_3d:
        # 2x2x2 neighborhood
        for x in range(image.shape[0] - 1):
            for y in range(image.shape[1] - 1):
                for z in range(image.shape[2] - 1):
                    # Extract 2x2x2 block
                    block = image[x:x+2, y:y+2, z:z+2]
                    
                    # Vertices: 8 vertices
                    for v in block.flatten():
                        H[v - min_val] += 1
                    
                    # Faces: 12 faces in 2x2x2 cube
                    # Each face is a 2x2 square
                    # Increment for min intensity of the 4 pixels in the face
                    faces = get_faces_2x2x2()
                    for face in faces:
                        min_face_val = np.min(block[face])
                        H[min_face_val - min_val] += 1
                        
                    # Cells: 1 voxel (the 2x2x2 block represents 1 cell in the dual sense? 
                    # Note: Source says "decrementing the histogram for 8-voxel-cubes in a 2x2x2 neighborhood"
                    # This implies the cell contribution is associated with the block itself)
                    min_block_val = np.min(block)
                    H[min_block_val - min_val] -= 1
                    
    else:
        # 2x2 neighborhood
        for x in range(image.shape[0] - 1):
            for y in range(image.shape[1] - 1):
                block = image[x:x+2, y:y+2]
                
                # Vertices: 4 vertices
                for v in block.flatten():
                    H[v - min_val] += 1
                    
                # Edges: 4 edges in 2x2 square
                # Decrement for min intensity of the 2 pixels in the edge
                edges = get_edges_2x2()
                for edge in edges:
                    min_edge_val = np.min(block[edge])
                    H[min_edge_val - min_val] -= 1
                    
    # Accumulate from highest bin downwards
    # H[T] should contain cumulative sum from max down to T
    for t in range(len(H) - 2, -1, -1):
        H[t] += H[t+1]
        
    # Compute TEC
    tec = np.sum(np.abs(H))
    return tec
```

## Subtopics

### Euler Characteristic from Mesh
A simple topological invariant calculation for a simplicial complex mesh $M$. It computes $\chi(M) = V(M) - E(M) + F(M)$, where $V, E, F$ are the number of vertices, edges, and triangles respectively [S1, unofficial]. This is used as the ground truth in the verification pipeline to compare against the SMT-computed $\chi$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   **Verification:** Depends on the number of critical points and the depth of recursion for link computation. The verification pipeline is repeated `#tests` times, making it a statistical verification tool rather than a real-time method [S1, unofficial].
    *   **Noise Estimation (TEC):** Computation cost is comparable to Total Variation (TV). In 2D, it samples 4 intensity points per pixel; in 3D, 8 points [S2, unofficial]. The histogram accumulation is negligible relative to the scan [S2, unofficial].
*   **Memory Complexity:**
    *   **Verification:** Requires storage for the grid and critical points.
    *   **Noise Estimation:** Requires a histogram array of size equal to the number of intensity levels (e.g., 256 for 8-bit) [S2, unofficial].
*   **Strengths:**
    *   **Verification:** Handles surfaces with boundary, which other topological verification methods (like Digital Surface tests) may struggle with without additional theory [S1, unofficial]. It is effective when used in unison with Digital Surface tests [S1, unofficial].
    *   **Noise Estimation:** Superior to TV and Total Second Derivatives (TSD) for scenes with many true gradients and curvatures, as TEC approximates zero for vanishing noise better than TV/TSD [S2, unofficial]. It is computationally efficient and exact for integer-valued data [S2, unofficial].
*   **Limitations:**
    *   **Verification:** Less sensitive than Digital Surface tests because it only provides $\chi$, making it harder to identify the source of failure [S1, unofficial]. It may fail to detect errors if the code introduces features that preserve $\chi$ (e.g., a torus plus cycling triangles) [S1, unofficial].
    *   **Noise Estimation:** Numerical inaccuracies at thin crests (e.g., Marschner-Lobb phantom) can produce speckles, causing TEC to not be exactly zero in the absence of noise [S2, unofficial]. Extending to floating-point data requires analysis of approximation error relative to histogram bin width [S2, unofficial].
*   **When to Use:**
    *   Use SMT Verification for debugging and verifying isosurface extraction algorithms that guarantee topology [S1, unofficial].
    *   Use TEC for automated evaluation of direct volume renderings and transfer function design, especially when distinguishing noise from true signal gradients is critical [S2, unofficial].

## Background
Stratified Morse Theory for Euler Characteristic was introduced to handle the non-smoothness of piecewise trilinear interpolants in isosurface extraction [S1, unofficial]. The verification pipeline (MMS-SMT) was developed to provide a robust method for verifying isosurfacing algorithms, particularly those handling ambiguous cases [S1, unofficial]. The application of Euler Characteristic as a noise measure (TEC) was presented by R. Wiemker in 2013, copyrighted by The Eurographics Association [S2, unofficial]. The method builds on classical Morse theory but adapts it to stratified spaces to handle critical points on vertices, edges, and faces of grid cells [S1, unofficial].

## Sources
[S1] Topology Verification for Isosurface Extraction, https://hal.science/hal-01211133, fetched 2026-10-26 (unofficial)
[S2] Total Euler Characteristic as a Noise Measure to aid Transfer Function Design, https://doi.org/10.2312/pe.eurovisshort.eurovisshort2013.097-101, fetched 2026-10-29 (unofficial)
