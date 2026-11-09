# Two-Step Surface Reconstruction Method

Also called: Improved signed marching cubes method, Signed Marching Cubes (SMC), Signed Marching Cubes with FMM, Two-Step Surface Reconstruction Method Using Signed Marching Cubes, Two-step reconstruction

## Overview
The Two-Step Surface Reconstruction Method is an efficient technique for extracting a target isosurface from an implicit function, specifically addressing the computational bottleneck of traditional Marching Cubes where every voxel in the domain must be evaluated [S1]. It accelerates reconstruction by filtering out invalid voxels (those not intersecting the isosurface) using a low-cost, low-accuracy evaluation, and then accurately evaluating only the valid boundary voxels [S1]. Unlike surface-following methods, it does not require seed voxels, ensuring the complete recovery of disconnected components [S1].

## How it works
The method combines Radial Basis Function (RBF) interpolation with the Fast Multipole Method (FMM) to evaluate the implicit function. It operates in two distinct evaluation phases: a coarse screening phase using a lower FMM expansion order and an exact evaluation phase using a higher expansion order.

### Inputs and Outputs
*   **Input:** A set of scattered interpolation points $\{x_1, ..., x_N\}$ with associated weights or signs, defining an implicit field.
*   **Output:** A triangular mesh representing the zero-level set ($S(x)=0$) of the implicit function.

### Data Structures
The method relies on the standard Marching Cubes lookup tables (`edgeTable`, `triangleTable`) as defined in [Marching Cubes Lookup and Configuration](52-marching-cubes-lookup-and-configuration.md).

1.  **Implicit Function Approximation:** The function $s(x)$ is represented as:
    $$ s(x) = \sum_{j=1}^{N} \omega_j \Phi(x, x_j) + p(x) $$
    where $\Phi$ is the radial basis kernel, $\omega_j$ are weights, and $p(x)$ is a polynomial part (e.g., $p(x) = c_1 + c_2 x + c_3 y + c_4 z$) [S1].

2.  **FMM Expansion:** The kernel $\Phi(x, y)$ is approximated using a finite series of order $p$:
    $$ \Phi(x, y) = \sum_{n=1}^{p} \phi_n(x)\psi_n(y) + R_p(x, y) $$
    where $R_p(x, y)$ is the residual error [S1]. The moments $\Psi_n(y) = \sum_{j=1}^N \omega_j \psi_n(y_j)$ are precomputed [S1].

3.  **Voxel Classification:** Voxels are classified into three sets:
    *   $C_{int}$: Interior voxels (sign is unambiguously negative).
    *   $C_{ext}$: Exterior voxels (sign is unambiguously positive).
    *   $C_{bou}$: Boundary voxels (sign is ambiguous or near zero).

### Parameters
*   **Voxel Size ($S_c$):** Determines reconstruction precision. It is derived from a reference length $l_0$ based on the bounding box dimensions $l_1, l_2, l_3$ (where $l_{10} < l_{20} < l_{30}$ are reference axes):
    $$ l_0 = \begin{cases} 0.1 l_2 & \text{if } l_3 < 0.1 l_2 \\ 0.05 l_3 & \text{if } 0.1 l_2 \le l_3 \le 0.5 l_2 \\ 0.025 l_3 & \text{if } l_3 > 0.5 l_2 \end{cases} $$
    Then, $S_c$ is set to $5l_0$ (coarse), $l_0$ (general), or $0.3l_0$ (fine) [S1].
*   **Lower Expansion Order ($p_0$):** Used for rough evaluation. Recommended range: 5–8 [S1].
*   **Higher Expansion Order ($p$):** Used for exact evaluation. Recommended range: 9–12 [S1].
*   **Evaluation Error ($\varepsilon_p$):** The residual error bound for expansion order $p$:
    $$ \varepsilon_p = \sum_{j=1}^{N} \omega_j R_p(x, y_j) $$
    [S1].

### Steps
1.  **Precomputation:** Precompute the FMM expansions for the RBF kernel using the black-box FMM for the specified orders $p_0$ and $p$ [S1].
2.  **Implicit Function Construction:** Solve the interpolation constraints to determine weights $\omega_j$ and polynomial coefficients, obtaining the implicit function $f(x)$ [S1].
3.  **Domain Discretization:** Determine the minimum bounding box of the sampling data. Divide this region into voxels $C$ with size $S_c$ [S1].
4.  **Rough Evaluation (Step 1):** Traverse all voxels $C$. For each vertex of each voxel, estimate the function value $s_{p_0}(x)$ using the FMM with the lower expansion order $p_0$ [S1].
5.  **Screening (Step 2):** Classify voxels based on $s_{p_0}(x)$ and the error bound $\varepsilon_{p_0}$:
    *   An exterior voxel is identified if $s_{p_0}(x) > 0$ and $s_{p_0}(x) - \varepsilon_{p_0} > 0$ [S1].
    *   An interior voxel is identified if $s_{p_0}(x) < 0$ and $s_{p_0}(x) + \varepsilon_{p_0} < 0$ [S1].
    *   Voxels not meeting these criteria are classified as boundary voxels $C_{bou}$ [S1].
    *   *Note:* The optimized $p_0$ satisfies $\varepsilon_{p_0-1} > |\sigma| > \varepsilon_{p_0} > \varepsilon_e$ where $\sigma$ is the accurate value [S1].
6.  **Exact Evaluation (Step 3):** Traverse only the boundary voxels $C_{bou}$. Compute accurate function values $s_p(x)$ for all 8 vertices using the FMM with the higher expansion order $p$ [S1].
7.  **Triangulation (Step 4):** Apply the standard Marching Cubes algorithm to each voxel in $C_{bou}$ using the accurate values and the lookup table to extract triangles [S1].
8.  **Mesh Assembly:** Unite all triangular facets to form the complete mesh [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_step_surface_reconstruction(interpolation_points, weights, kernel_function, 
                                    polynomial_basis, p_low, p_high, voxel_size):
    """
    Implements the Two-Step Surface Reconstruction Method.
    
    Args:
        interpolation_points: List of 3D points {x_j}.
        weights: List of weights {omega_j}.
        kernel_function: The radial basis kernel Phi(x, y).
        polynomial_basis: Basis functions for p(x).
        p_low: Lower expansion order for screening.
        p_high: Higher expansion order for exact evaluation.
        voxel_size: Size of the voxel grid S_c.
        
    Returns:
        List of triangles forming the isosurface.
    """
    
    # Step 1: Construct Implicit Function
    # Solve for weights and polynomial coefficients to satisfy interpolation constraints.
    # This involves solving a linear system derived from the RBF interpolation equations.
    # s(x) = sum(omega_j * Phi(x, x_j)) + p(x)
    implicit_model = construct_rbf_model(interpolation_points, weights, kernel_function, polynomial_basis)
    
    # Step 2: Precompute FMM Expansions
    # Precompute moments and expansions for p_low and p_high using Black-Box FMM.
    fmm_low = BlackBoxFMM(kernel_function, expansion_order=p_low)
    fmm_high = BlackBoxFMM(kernel_function, expansion_order=p_high)
    
    # Step 3: Define Reconstruction Domain
    # Determine minimum bounding box of interpolation_points.
    bounding_box = get_min_bounding_box(interpolation_points)
    voxel_grid = create_voxel_grid(bounding_box, voxel_size)
    
    # Step 4: Rough Evaluation and Screening
    boundary_voxels = []
    
    for voxel in voxel_grid:
        # Get 8 vertices of the voxel
        vertices = voxel.get_vertices()
        
        # Evaluate all vertices using low-order FMM
        low_values = []
        for v in vertices:
            val = fmm_low.evaluate(v, implicit_model)
            low_values.append(val)
            
        # Get error bound for p_low
        epsilon_low = fmm_low.get_error_bound()
        
        # Classify voxel
        is_interior = True
        is_exterior = True
        
        for val in low_values:
            # Check for Interior: val < 0 and val + epsilon < 0
            if not (val < 0 and val + epsilon_low < 0):
                is_interior = False
                
            # Check for Exterior: val > 0 and val - epsilon > 0
            if not (val > 0 and val - epsilon_low > 0):
                is_exterior = False
                
        if is_interior:
            continue # Skip interior voxel
        elif is_exterior:
            continue # Skip exterior voxel
        else:
            # Ambiguous sign or near zero: Keep as boundary voxel
            boundary_voxels.append(voxel)
            
    # Step 5: Exact Evaluation
    accurate_values_map = {}
    
    for voxel in boundary_voxels:
        vertices = voxel.get_vertices()
        accurate_vals = []
        
        for v in vertices:
            # Evaluate using high-order FMM
            val = fmm_high.evaluate(v, implicit_model)
            accurate_vals.append(val)
            
        # Store accurate values for this voxel
        accurate_values_map[voxel.id] = accurate_vals
        
    # Step 6: Marching Cubes Triangulation
    triangles = []
    
    for voxel in boundary_voxels:
        vals = accurate_values_map[voxel.id]
        
        # Apply standard Marching Cubes algorithm
        # Uses edgeTable and triangleTable from [Marching Cubes Lookup and Configuration](52-marching-cubes-lookup-and-configuration.md)
        voxel_triangles = marching_cubes_step(vals, voxel.vertices, voxel_size)
        triangles.extend(voxel_triangles)
        
    return triangles

def construct_rbf_model(points, weights, kernel, poly_basis):
    """
    Solves the RBF interpolation system.
    Returns an object capable of evaluating s(x).
    """
    # Implementation detail: Solve linear system A * [omega; c] = b
    # where A contains kernel evaluations and polynomial constraints.
    # This is a standard linear algebra operation.
    pass

class BlackBoxFMM:
    def __init__(self, kernel, expansion_order):
        self.kernel = kernel
        self.order = expansion_order
        # Precompute moments and multipole expansions here
        pass
        
    def evaluate(self, point, model):
        # Use FMM to approximate sum(omega_j * Phi(point, x_j))
        # Returns the approximate scalar value
        pass
        
    def get_error_bound(self):
        # Returns epsilon_p for the current expansion order
        pass
```

### Edge Cases and Pitfalls
*   **Ambiguous Sign:** If the rough evaluation $s_{p_0}(x)$ is close to zero such that $s_{p_0}(x) - \varepsilon_{p_0} \le 0$ (for positive check) or $s_{p_0}(x) + \varepsilon_{p_0} \ge 0$ (for negative check), the sign is ambiguous. These voxels are treated as boundary voxels and undergo exact evaluation [S1].
*   **Small Objects:** If the target object size is smaller than the specified voxel size $S_c$, the object may not intersect any voxels, leading to failure to detect it [S1]. This requires specifying a smaller voxel size [S1].
*   **Sharp Features:** Like standard Marching Cubes, the method struggles to recover sharp geometric features of the target isosurface [S1].
*   **Seed Dependency:** Unlike Voxel Growing, this method does not depend on seed voxels, robustly handling separated geometry parts [S1].

## Operating characteristics
*   **Time Complexity:** Traditional Marching Cubes requires $O(N)$ evaluations per point for direct RBF evaluation, leading to high costs for large domains. The Two-Step method reduces total time by avoiding high-precision FMM (high $p$) for non-boundary voxels. The number of high-precision evaluations is significantly reduced [S1].
*   **Measured Performance:** In tests with $N=10,704$ points and $S_c=5$, the ratio of high-precision evaluations to total evaluations was 1.92% [S1]. For $N=6,999$ and $S_c=2$, the ratio was 22.53% [S1]. Efficiency improvements are more pronounced with smaller voxel sizes [S1].
*   **Memory Complexity:** Requires storage for the FMM data structures (moments, expansions) and the voxel grid. The memory overhead is dominated by the FMM precomputation and the list of boundary voxels, which is typically much smaller than the total voxel count [S1].
*   **Strengths:** Eliminates the need for seed voxels; guarantees complete reconstruction of disconnected components; significantly reduces computation time for large sparse domains compared to full-domain Marching Cubes [S1].
*   **Limitations:** Cannot detect objects smaller than the voxel size; limited ability to capture sharp features; accuracy depends on the choice of expansion orders $p_0$ and $p$ [S1].

## Background
The method was presented by J. Zhang and D. Zhong in *Applied Sciences* (2022) [S1]. It builds upon the principles of Signed Marching Cubes and the Fast Multipole Method to accelerate RBF-based implicit surface reconstruction. It addresses the limitations of traditional Marching Cubes (computational cost) and surface-following methods (seed dependency) [S1]. It has been implemented using the Scal-FMM library for single-point FMM and VTK for visualization [S1].

## Sources
[S1] A Two-Step Surface Reconstruction Method Using Signed Marching Cubes, https://doi.org/10.3390/app12041792, fetched 2026-10-27
