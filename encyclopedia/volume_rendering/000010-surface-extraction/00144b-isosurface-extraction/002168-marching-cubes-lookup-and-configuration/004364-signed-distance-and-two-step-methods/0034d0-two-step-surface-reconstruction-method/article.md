# Two-Step Surface Reconstruction Method

Also called: Improved signed marching cubes method, Signed Marching Cubes (SMC), Signed Marching Cubes with FMM, Two-Step Surface Reconstruction Method Using Signed Marching Cubes, Two-step reconstruction

## Overview
The Two-Step Surface Reconstruction Method is an efficient technique for extracting a target isosurface from an implicit function defined by scattered data, typically via Radial Basis Function (RBF) interpolation. It solves the performance bottleneck of traditional Marching Cubes, which evaluates the implicit function for every voxel in the domain regardless of whether the voxel intersects the surface. By combining a low-cost rough evaluation using the Fast Multipole Method (FMM) with a lower expansion order to filter out non-boundary voxels, and a high-precision exact evaluation using a higher expansion order only for the remaining boundary voxels, the method significantly reduces computational cost while guaranteeing the completeness of the reconstructed surface without requiring seed voxels [S1].

## How it works
The method operates on an implicit function $f(x)$ constructed from scattered interpolation points. The core innovation is the separation of the evaluation process into two stages based on the distance of the voxel from the isosurface, utilizing the error bounds of the FMM approximation to determine validity.

### Inputs and Outputs
**Inputs:**
*   A set of $N$ interpolation points $\{x_1, \dots, x_N\}$ with associated weights $\omega_i$ and signs (interior, exterior, or on-surface).
*   Reconstruction parameters: Voxel size $S_c$, lower expansion order $p_0$, and higher expansion order $p$.
*   Target isovalue (typically 0 for the zero-level set).

**Outputs:**
*   A triangular mesh representing the isosurface.

### Data Structures
1.  **Implicit Function Model**: Defined by RBF interpolation.
    *   Weights $\omega_i$ for $i=1 \dots N$.
    *   Polynomial coefficients $c_1, c_2, c_3, c_4$ for the linear part $p(x) = c_1 + c_2 x + c_3 y + c_4 z$.
2.  **Voxel Grid**:
    *   A regular grid covering the minimum bounding box of the input data.
    *   Each voxel $C$ is identified by its index or corner coordinates.
    *   Classification flags for each voxel: `Interior`, `Exterior`, or `Boundary`.
3.  **FMM Precomputed Moments**:
    *   For each expansion order $k \in \{p_0, p\}$, precomputed moments $\Psi_n(y)$ for $n=1 \dots p_{max}$, where $p_{max}$ is the highest order used.

### Formulas
The implicit function $s(x)$ is defined as [S1]:
$$ s(x) = \sum_{j=1}^{N} \omega_j \Phi(x, x_j) + p(x) $$
where $\Phi$ is the radial basis kernel (e.g., biharmonic or triharmonic spline) and $p(x)$ is a first-degree polynomial satisfying $\sum \omega_i = \sum \omega_i x_i = \sum \omega_i y_i = \sum \omega_i z_i = 0$ [S1].

The FMM approximates the kernel $\Phi(x, y)$ using a finite series [S1]:
$$ \Phi(x, y) = \sum_{n=1}^{p} \phi_n(x) \psi_n(y) + R_p(x, y) $$
The approximate function value $s_p(x)$ is computed as [S1]:
$$ s_p(x) = \sum_{n=1}^{p} \Psi_n(y) \phi_n(x) + p(x) $$
where $\Psi_n(y) = \sum_{j=1}^{N} \omega_j \psi_n(y_j)$ are the precomputed moments.

The evaluation error $\varepsilon_p$ is defined as [S1]:
$$ \varepsilon_p = \sum_{j=1}^{N} |\omega_j R_p(x, y_j)| $$
*Note: The source defines $\varepsilon_p = \sum \omega_j R_p(x, y_j)$ but implies magnitude for error bounds. In implementation, $\varepsilon_p$ represents the upper bound of the residual error.*

### Parameters
*   **Voxel Size ($S_c$)**: Determined by the reference length $l_0$ and desired accuracy [S1]:
    *   Coarse: $S_c = 5 l_0$
    *   General: $S_c = l_0$
    *   Fine: $S_c = 0.3 l_0$
*   **Reference Length ($l_0$)**: Derived from the minimum bounding box of the data with axis lengths $l_1 \le l_2 \le l_3$ [S1]:
    *   If $l_3 < 0.1 l_2$: $l_0 = 0.1 l_2$
    *   If $0.1 l_2 \le l_3 \le 0.5 l_2$: $l_0 = 0.05 l_3$
    *   If $l_3 > 0.5 l_2$: $l_0 = 0.025 l_3$
*   **Expansion Orders**:
    *   Lower order $p_0$: Recommended 5–8.
    *   Higher order $p$: Recommended 9–12.

### Algorithm Steps

1.  **Preprocessing**:
    *   Compute the minimum bounding box of the interpolation points.
    *   Calculate $l_0$ and $S_c$ based on the bounding box dimensions and desired accuracy.
    *   Solve the RBF interpolation system to obtain weights $\omega_i$ and polynomial coefficients.
    *   Precompute FMM moments $\Psi_n$ for expansion orders $p_0$ and $p$.

2.  **Rough Evaluation (Step 1)**:
    *   Traverse all voxels $C$ in the grid.
    *   For each vertex $v$ of each voxel, evaluate the approximate function $s_{p_0}(v)$ using the lower expansion order $p_0$.
    *   Calculate the error bound $\varepsilon_{p_0}$ for this evaluation.

3.  **Screening (Step 2)**:
    *   Classify each voxel based on the signs of its vertices evaluated with $p_0$.
    *   For a vertex value $\sigma = s_{p_0}(v)$:
        *   If $\sigma > 0$ and $\sigma - \varepsilon_{p_0} > 0$, the sign is reliably positive.
        *   If $\sigma < 0$ and $\sigma + \varepsilon_{p_0} < 0$, the sign is reliably negative.
        *   If neither condition is met (i.e., $|\sigma| \le \varepsilon_{p_0}$ or the sign is ambiguous relative to the error), the sign is ambiguous.
    *   A voxel is classified as **Boundary** ($C_{bou}$) if any of its vertices have an ambiguous sign or if the voxel contains vertices with conflicting reliable signs (indicating the isosurface passes through).
    *   Voxels where all vertices are reliably positive are **Exterior** ($C_{ext}$).
    *   Voxels where all vertices are reliably negative are **Interior** ($C_{int}$).
    *   Discard $C_{int}$ and $C_{ext}$.

4.  **Exact Evaluation (Step 3)**:
    *   Traverse only the voxels in $C_{bou}$.
    *   For each vertex of these voxels, evaluate the function $s_p(v)$ using the higher expansion order $p$.
    *   This provides high-accuracy function values for the Marching Cubes lookup.

5.  **Marching Cubes (Step 4)**:
    *   For each voxel in $C_{bou}$, use the accurate values $s_p(v)$ at its 8 vertices.
    *   Apply the standard Marching Cubes lookup table to determine the intersection topology with the isosurface (isovalue 0).
    *   Generate triangles for the intersections.

6.  **Mesh Assembly**:
    *   Unite all generated triangular facets into a single mesh.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_step_surface_reconstruction(interpolation_points, voxel_size, p_low, p_high):
    """
    interpolation_points: List of (x, y, z, weight, sign)
    voxel_size: Sc
    p_low: p0
    p_high: p
    """
    # Step 1: Preprocessing
    bounding_box = compute_min_max(interpolation_points)
    l1, l2, l3 = bounding_box.dimensions()
    l0 = compute_reference_length(l1, l2, l3)
    # If voxel_size is not explicitly provided, compute it:
    # voxel_size = compute_voxel_size(l0, accuracy_level) 
    
    # Solve RBF interpolation to get weights and polynomial coeffs
    weights, poly_coeffs = solve_rbf_system(interpolation_points)
    
    # Precompute FMM moments for both orders
    moments_low = precompute_fmm_moments(interpolation_points, weights, p_low)
    moments_high = precompute_fmm_moments(interpolation_points, weights, p_high)
    
    # Define grid
    grid = create_grid(bounding_box, voxel_size)
    
    boundary_voxels = []
    
    # Step 2 & 3: Rough Evaluation and Screening
    for voxel in grid:
        vertex_values_low = []
        vertex_errors_low = []
        
        for vertex in voxel.vertices:
            # Evaluate with low order
            val = evaluate_fmm_approx(vertex, moments_low, poly_coeffs)
            err = compute_error_bound(vertex, interpolation_points, weights, p_low)
            vertex_values_low.append(val)
            vertex_errors_low.append(err)
            
        # Classify voxel
        is_boundary = False
        for val, err in zip(vertex_values_low, vertex_errors_low):
            # Check for ambiguous sign
            # Positive check: val > 0 AND val - err > 0
            # Negative check: val < 0 AND val + err < 0
            # If neither, sign is ambiguous -> Boundary
            if not (val > 0 and val - err > 0) and not (val < 0 and val + err < 0):
                is_boundary = True
                break
        
        # Also check for sign changes among reliable vertices
        reliable_signs = []
        for val, err in zip(vertex_values_low, vertex_errors_low):
            if val > 0 and val - err > 0:
                reliable_signs.append(1)
            elif val < 0 and val + err < 0:
                reliable_signs.append(-1)
            else:
                reliable_signs.append(0) # Ambiguous
                
        if 1 in reliable_signs and -1 in reliable_signs:
            is_boundary = True
            
        if is_boundary:
            boundary_voxels.append(voxel)
            
    # Step 4: Exact Evaluation
    accurate_values = {} # Map voxel_id -> list of 8 vertex values
    for voxel in boundary_voxels:
        vals = []
        for vertex in voxel.vertices:
            val = evaluate_fmm_approx(vertex, moments_high, poly_coeffs)
            vals.append(val)
        accurate_values[voxel.id] = vals
        
    # Step 5: Marching Cubes
    triangles = []
    for voxel in boundary_voxels:
        vals = accurate_values[voxel.id]
        # Apply standard Marching Cubes lookup table
        # Input: vals (8 floats), voxel geometry
        # Output: list of triangles
        mesh_fragments = marching_cubes_lookup(vals, voxel)
        triangles.extend(mesh_fragments)
        
    return assemble_mesh(triangles)

def compute_reference_length(l1, l2, l3):
    # Assume l1 <= l2 <= l3
    if l3 < 0.1 * l2:
        return 0.1 * l2
    elif 0.1 * l2 <= l3 <= 0.5 * l2:
        return 0.05 * l3
    else:
        return 0.025 * l3

def evaluate_fmm_approx(point, moments, poly_coeffs):
    # phi_n(point) * sum(moments) + poly(point)
    # Simplified: The library handles the basis functions phi_n
    return fmm_evaluate(point, moments) + evaluate_polynomial(point, poly_coeffs)

def compute_error_bound(point, points, weights, order):
    # Sum of |omega_j * R_p(x, y_j)|
    # R_p is the residual of the kernel expansion
    # In practice, this is often provided by the FMM library or approximated
    return fmm_error_bound(point, points, weights, order)
```

### Edge Cases and Pitfalls
*   **Ambiguous Sign**: If the rough evaluation $s_{p_0}(x)$ is close to zero such that $s_{p_0}(x) - \varepsilon_{p_0} \le 0$ (for positive check) or $s_{p_0}(x) + \varepsilon_{p_0} \ge 0$ (for negative check), the sign cannot be determined reliably. These voxels are treated as boundary voxels requiring exact evaluation [S1].
*   **Small Objects**: If the target object size is smaller than the specified reconstruction voxel size $S_c$, the object cannot be detected, leading to failure reconstruction. A smaller voxel size must be specified to recover undetected objects [S1].
*   **Sharp Features**: Like standard Marching Cubes, the method may fail to recover sharp geometric features of the target isosurface [S1].
*   **Seed Dependency**: Unlike voxel growing methods, this method does not depend on initial seed voxels, ensuring complete recovery even for disconnected components [S1].

## Operating characteristics
*   **Time Complexity**: Traditional Marching Cubes with direct RBF evaluation is $O(N)$ per point, leading to $O(N \cdot V)$ total time for $V$ voxels. The Two-Step method reduces the number of high-precision evaluations. The rough evaluation is $O(V)$ with low cost, and the exact evaluation is $O(V_{bou})$ with high cost, where $V_{bou} \ll V$. The total time is significantly less than traditional MC for large volumes with sparse surfaces [S1].
*   **Memory Complexity**: Requires storage for the voxel grid, the implicit function weights, and FMM moments. Memory usage is dominated by the FMM data structures and the grid, similar to standard MC but with additional overhead for precomputed moments [S1].
*   **Measured Performance**: In experiments with $N=10,704$ points and voxel size $S_c=5$, the number of high-precision evaluations was only 1.92% of the total voxels, while low-precision evaluations covered 96% of voxels. For $N=6,999$ and $S_c=2$, high-precision evaluations were 22.53% of voxels. The method showed superior efficiency compared to traditional MC, with efficiency gains increasing as voxel size decreased [S1].
*   **Strengths**: Eliminates the need for seed voxels; guarantees completeness of disconnected components; significantly faster than traditional MC for implicit surfaces defined by RBFs.
*   **Limitations**: Cannot detect objects smaller than the voxel size; struggles with sharp features; requires careful tuning of expansion orders and voxel size.

## Background
The method was presented by J. Zhang and D. Zhong in *Applied Sciences* (2022) [S1]. It builds upon the Signed Marching Cubes (SMC) concept, which uses the sign of the implicit function to guide surface extraction. The integration of the Fast Multipole Method (FMM) allows for the efficient evaluation of the RBF-based implicit functions, which are otherwise computationally expensive. The two-step approach specifically addresses the inefficiency of evaluating all voxels with high precision by leveraging the error bounds of the FMM approximation to filter non-boundary voxels. The implementation in the source paper used the Scal-FMM library for FMM operations and VTK for visualization [S1].

## Sources
[S1] A Two-Step Surface Reconstruction Method Using Signed Marching Cubes, https://doi.org/10.3390/app12041792, fetched 2026-10-27
