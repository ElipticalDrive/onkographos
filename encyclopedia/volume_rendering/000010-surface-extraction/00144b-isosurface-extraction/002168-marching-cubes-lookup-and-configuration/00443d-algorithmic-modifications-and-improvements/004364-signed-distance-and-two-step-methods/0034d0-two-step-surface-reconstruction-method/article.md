# Two-Step Surface Reconstruction Method

Also called: Improved signed marching cubes method, Signed Marching Cubes (SMC), Signed Marching Cubes with FMM, Two-Step Surface Reconstruction Method Using Signed Marching Cubes, Two-step reconstruction

## Overview
The Two-Step Surface Reconstruction Method accelerates isosurface extraction from implicit functions by filtering non-intersecting voxels using a low-cost rough evaluation before performing accurate evaluation only on potential boundary voxels [S1]. It solves the excessive computation time of traditional Marching Cubes, which evaluates all voxels regardless of their proximity to the surface [S1]. Unlike surface-following methods that require seed voxels and may miss disconnected components, this method guarantees complete isosurface recovery without seed dependencies [S1].

## How it works
**Inputs:**
*   A set of $N$ interpolation points $\{x_1, ..., x_N\}$ with associated signs (interior, exterior, or on surface) or scalar values.
*   Reconstruction parameters: Voxel size $S_c$, lower expansion order $p_0$, higher expansion order $p$.

**Outputs:**
*   A triangular mesh representing the target isosurface.

**Data Structures:**
*   **Implicit Function:** Defined by Radial Basis Function (RBF) interpolation:
    $$s(x) = \sum_{j=1}^{N} \omega_j \Phi(x, x_j) + p(x)$$
    where $\Phi$ is the radial basis kernel, $\omega_j$ are weight coefficients, and $p(x)$ is a polynomial part (e.g., $p(x) = c_1 + c_2 x + c_3 y + c_4 z$) [S1].
*   **FMM Approximation:** The kernel $\Phi(x, y)$ is approximated using a finite series of order $p$:
    $$\Phi(x, y) = \sum_{n=1}^{p} \phi_n(x)\psi_n(y) + R_p(x, y)$$
    where $R_p(x, y)$ is the residual [S1].
*   **Estimating Function:** The approximated function value at point $x$ using order $p$ is:
    $$s_p(x) = \sum_{n=1}^{p} \Psi_n(y)\phi_n(x)$$
    where $\Psi_n(y) = \sum_{j=1}^{N} \omega_j \psi_n(y_j)$ are pre-computed moments [S1].
*   **Evaluation Error:** The bound for the approximation error is:
    $$\varepsilon_p = \sum_{j=1}^{N} \omega_j R_p(x, y_j)$$
    [S1].
*   **Voxel Grid:** A structured 3D grid covering the minimum bounding box of the sampling data, divided into voxels $C$ with size $S_c$ [S1].

**Algorithm Steps:**

1.  **Parameter Specification:**
    Determine the voxel size $S_c$ based on the model's minimum bounding box dimensions ($l_1, l_2, l_3$). Calculate a reference length $l_0$ based on the relative lengths of the bounding box axes [S1]:
    $$
    l_0 = \begin{cases} 
    0.1 l_2 & \text{if } l_3 < 0.1 l_2 \\
    0.05 l_3 & \text{if } 0.1 l_2 \le l_3 \le 0.5 l_2 \\
    0.025 l_3 & \text{if } l_3 > 0.5 l_2 
    \end{cases}
    $$
    Then set $S_c$ according to desired accuracy: coarse ($S_c = 5l_0$), general ($S_c = l_0$), or fine ($S_c = 0.3l_0$) [S1]. Select $p_0$ (lower order) and $p$ (higher order) [S1].

2.  **Precomputation:**
    Precompute the moments $\Psi_n(y)$ for both expansion orders $p_0$ and $p$ using the black-box Fast Multipole Method (FMM) to accelerate RBF evaluation [S1].

3.  **Implicit Function Construction:**
    Solve the interpolation constraints from the sampling data to determine weights $\omega_j$ and polynomial coefficients, defining the implicit function $f(x)$ [S1].

4.  **Grid Generation:**
    Determine the reconstruction region using the minimum bounding box of the sampling data. Divide this region into voxels $C$ [S1].

5.  **Rough Evaluation (Step 1):**
    Traverse all voxels $C$. For each vertex of each voxel, estimate the sign of the function value using the estimating function $s_{p_0}(x)$ with the lower expansion order $p_0$ [S1].

6.  **Screening (Step 2):**
    Classify voxels based on the rough evaluation results. A voxel is classified as:
    *   **Exterior ($C_{ext}$):** If for all vertices, $s_{p_0}(x) = \sigma > 0$ and $s_{p_0}(x) - \varepsilon_{p_0} > 0$. This requires $\varepsilon_{p_0-1} > \sigma > \varepsilon_{p_0} > \varepsilon_e$ [S1].
    *   **Interior ($C_{int}$):** If for all vertices, $s_{p_0}(x) = \sigma < 0$ and $s_{p_0}(x) + \varepsilon_{p_0} < 0$. This requires $\varepsilon_{p_0-1} > |\sigma| > \varepsilon_{p_0} > \varepsilon_e$ [S1].
    *   **Boundary ($C_{bou}$):** If the sign is ambiguous (i.e., $s_{p_0}(x) - \varepsilon_{p_0} \le 0$ for positive check or $s_{p_0}(x) + \varepsilon_{p_0} \ge 0$ for negative check) or the value is near zero [S1].
    Eliminate $C_{ext}$ and $C_{int}$ to obtain the set of possible boundary voxels $C_{bou}$ [S1].

7.  **Exact Evaluation (Step 3):**
    Traverse only the voxels in $C_{bou}$. Compute accurate function values for the vertices using the estimating function $s_p(x)$ with the higher expansion order $p$ [S1].

8.  **Marching Cubes (Step 4):**
    Apply the standard Marching Cubes algorithm to each boundary voxel in $C_{bou}$ using the accurate function values to determine edge intersections and generate triangles [S1].

9.  **Mesh Assembly:**
    Unite the triangular facets from all boundary voxels to obtain the complete triangular mesh [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_step_surface_reconstruction(interpolation_points, signs, voxel_size, p_low, p_high):
    """
    interpolation_points: List of 3D coordinates [x, y, z]
    signs: List of integers (-1 for interior, 1 for exterior, 0 for on surface)
    voxel_size: Float, size of the voxel grid cells
    p_low: Integer, lower expansion order for rough evaluation
    p_high: Integer, higher expansion order for exact evaluation
    """
    # Step 1 & 2: Precompute FMM moments for both orders
    # This assumes a Black-Box FMM implementation is available
    moments_low = precompute_fmm_moments(interpolation_points, signs, p_low)
    moments_high = precompute_fmm_moments(interpolation_points, signs, p_high)
    
    # Step 3 & 4: Define grid based on bounding box of interpolation_points
    # Determine bounding box min/max
    min_coords = min(p[:3] for p in interpolation_points)
    max_coords = max(p[:3] for p in interpolation_points)
    
    # Generate voxel grid indices covering the bounding box
    # Grid spacing is voxel_size
    grid = generate_voxel_grid(min_coords, max_coords, voxel_size)
    
    boundary_voxels = []
    
    # Step 5 & 6: Rough Evaluation and Screening
    for voxel in grid:
        vertices = voxel.get_vertices() # Returns 8 corner coordinates
        is_boundary = False
        
        for vertex in vertices:
            # Evaluate using lower order
            val_low = evaluate_fmm_approx(vertex, moments_low)
            err_low = compute_error_bound(vertex, moments_low) # epsilon_p0
            
            # Check if sign is unambiguous and outside error bound
            # We need to know the true sign context, but here we check if 
            # the value is confidently positive or negative relative to 0.
            # If val_low > err_low, it's confidently positive.
            # If val_low < -err_low, it's confidently negative.
            # If |val_low| <= err_low, it's ambiguous.
            
            if abs(val_low) <= err_low:
                is_boundary = True
                break
            # Note: The paper defines classification based on specific inequalities
            # involving sigma and epsilon. If the voxel is entirely exterior 
            # (all vertices > err) or interior (all vertices < -err), skip.
            # If any vertex is ambiguous, the voxel is a boundary candidate.
        
        if is_boundary:
            boundary_voxels.append(voxel)
            
    # Step 7: Exact Evaluation
    accurate_values = {}
    for voxel in boundary_voxels:
        vertices = voxel.get_vertices()
        for vertex in vertices:
            # Evaluate using higher order
            val_high = evaluate_fmm_approx(vertex, moments_high)
            accurate_values[vertex] = val_high
            
    # Step 8 & 9: Marching Cubes and Assembly
    mesh_triangles = []
    for voxel in boundary_voxels:
        # Get accurate values for the 8 vertices of this voxel
        vertex_values = [accurate_values[v] for v in voxel.get_vertices()]
        
        # Apply standard Marching Cubes lookup
        # Case index is determined by signs of vertex_values relative to 0
        case_index = compute_marching_cubes_case_index(vertex_values)
        
        # Generate triangles based on case_index and vertex positions
        triangles = generate_triangles_from_case(case_index, voxel.get_vertices())
        mesh_triangles.extend(triangles)
        
    return mesh_triangles

def precompute_fmm_moments(points, signs, order):
    """
    Placeholder for Black-Box FMM moment computation.
    Returns moments Psi_n for the given expansion order.
    """
    # In a real implementation, this calls the FMM library (e.g., Scal-FMM)
    # to compute the multipole expansions.
    pass

def evaluate_fmm_approx(point, moments):
    """
    Evaluates the approximated implicit function s_p(x) at a point
    using precomputed moments.
    """
    # Sum over n=1 to p: Psi_n * phi_n(point)
    # This requires the basis functions phi_n to be defined.
    pass

def compute_error_bound(point, moments):
    """
    Computes the evaluation error epsilon_p.
    """
    # Sum of omega_j * R_p(x, y_j)
    # This is the residual bound.
    pass
```

**Parameters:**
*   **Expansion Order ($p$):** The order of the finite series in FMM. Higher $p$ increases accuracy but consumes more time. Recommended $p$ (higher) is 9–12. Recommended $p_0$ (lower) is 5–8 [S1].
*   **Voxel Size ($S_c$):** Determines reconstruction precision. Coarse: $5l_0$, General: $l_0$, Fine: $0.3l_0$ [S1].
*   **Evaluation Error ($\varepsilon_p$):** The residual error of the FMM approximation, used as a threshold for sign determination [S1].

**Edge Cases and Pitfalls:**
*   **Ambiguous Sign:** If the rough evaluation $s_{p_0}(x)$ is close to zero such that $|s_{p_0}(x)| \le \varepsilon_{p_0}$, the sign is ambiguous. These voxels are treated as boundary voxels requiring exact evaluation [S1].
*   **Small Objects:** If the target object size is smaller than the specified voxel size $S_c$, the object may not intersect any voxels, leading to failure. A smaller $S_c$ is required [S1].
*   **Sharp Features:** Like standard Marching Cubes, this method may struggle to recover sharp geometric features [S1].
*   **Seed Dependency:** Unlike voxel growing, this method does not depend on seed voxels, ensuring complete recovery of disconnected components [S1].

## Subtopics

### SDF to PLY Conversion
Converts Signed Distance Fields directly to PLY meshes, often using narrow-band techniques. [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)

### Marching Cubes with Narrow-Band Filtering
Filters voxels based on distance to the isosurface to reduce computation. [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md)

## Operating characteristics
*   **Time Complexity:** Traditional Marching Cubes requires $O(N)$ operations per point for direct RBF evaluation. The Two-Step method reduces total time by avoiding high-precision FMM (high $p$) for non-boundary voxels. The number of evaluations with the higher expansion order is significantly reduced compared to the total number of voxels [S1].
*   **Measured Performance:** In tests with $N=10,704$ points and $S_c=5$, the ratio of high-precision evaluations to total evaluations was 1.92%. For $N=6,999$ and $S_c=2$, the ratio was 22.53%. Efficiency improvements were more obvious with smaller voxel sizes [S1].
*   **Strengths:** Eliminates invalid evaluation of non-boundary voxels; does not require seed voxels; guarantees complete isosurface recovery for disconnected parts [S1].
*   **Limitations:** May fail to detect objects smaller than the voxel size; struggles with sharp geometry features; requires careful selection of expansion orders and voxel sizes [S1].
*   **When to Use:** When the implicit function is expensive to evaluate (e.g., RBF) and the surface is sparse relative to the domain.
*   **When Not to Use:** When the surface is dense or fills most of the domain, or when sharp features are critical and voxel size cannot be reduced sufficiently.

## Background
The method was presented by J. Zhang and D. Zhong in *Applied Sciences* in 2022 [S1]. It builds on the Signed Marching Cubes (SMC) concept, which uses FMM to accelerate RBF evaluation. The core innovation is the two-step filtering process using different expansion orders to distinguish boundary from non-boundary voxels [S1]. It is implemented using the Scal-FMM library for FMM operations and VTK for visualization [S1].

## Sources
[S1] A Two-Step Surface Reconstruction Method Using Signed Marching Cubes, https://doi.org/10.3390/app12041792, fetched 2026-10-27
