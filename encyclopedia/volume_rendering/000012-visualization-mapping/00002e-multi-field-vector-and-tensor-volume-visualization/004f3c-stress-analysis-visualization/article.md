# Stress Analysis Visualization

Also called: Linear Elasticity Stress Visualization, Von Mises Stress Visualization

## Overview
Stress Analysis Visualization renders the internal mechanical stress state of a deformed object, typically derived from linear elasticity simulations [S1]. It solves the problem of visualizing scalar stress metrics (such as Von Mises stress) computed from tensor fields that are often not stored directly but must be derived from displacement fields on the fly [S1]. It is distinguished from general tensor visualization by its specific reliance on on-the-fly derivation of strain and stress tensors from displacement data in isogeometric analysis contexts, rather than pre-computed cell-centered values [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A displacement field $u: P \rightarrow \mathbb{R}^3$ defined over a parametric domain $P$, typically represented as cubic B-splines or similar basis functions in isogeometric analysis [S1].
*   **Output:** A volume-rendered image where color/opacity maps to the Von Mises stress scalar $\rho$ [S1].

**Data Structures:**
*   **Displacement Field:** Defined by control points and basis functions (e.g., cubic B-splines). The specific topology (e.g., volume blocks with control points) is defined by the simulation mesh [S1].
*   **Stress Tensor:** Computed per sample point. In some pipelines (e.g., physiological mesh data), stress is stored as point-associated data rather than cell-associated data [S2, unofficial].

**Formulas:**
1.  **Strain Tensor Calculation:**
    The strain tensor $\sigma$ is calculated using the Jacobian of the deformation. Let $J_u$ be the Jacobian of the displacement field $u$ and $J_\phi$ be the Jacobian of the geometry parametrization $\phi$. The term $J_{u \circ \phi^{-1}}$ is the solution to the linear system:
    $$ [J_\phi(p)]^T J_{u \circ \phi^{-1}}(g) = [J_u(p)]^T $$
    Then, the strain tensor is:
    $$ \sigma = \frac{1}{2} \left( J_{u \circ \phi^{-1}} + (J_{u \circ \phi^{-1}})^T \right) $$
    [S1]

2.  **Von Mises Stress:**
    The scalar Von Mises stress $\rho$ is derived from the components of the strain/stress tensor $\sigma$:
    $$ \rho = \frac{(\sigma_{11} - \sigma_{22})^2 + (\sigma_{22} - \sigma_{33})^2 + (\sigma_{33} - \sigma_{11})^2 + 6(\sigma_{12}^2 + \sigma_{23}^2 + \sigma_{31}^2)}{2} $$
    [S1]

**Steps:**
1.  **Sample:** For each sample point $\mathbf{x}$ in the volume rendering integral, evaluate the displacement field $u(\mathbf{x})$ and its Jacobian $J_u(\mathbf{x})$, and the geometry parametrization Jacobian $J_\phi(\mathbf{x})$ [S1].
2.  **Solve for Strain:** Solve the linear system $[J_\phi(\mathbf{x})]^T \mathbf{y} = [J_u(\mathbf{x})]^T$ for $\mathbf{y}$ (representing $J_{u \circ \phi^{-1}}$) [S1].
3.  **Compute Strain Tensor:** Calculate $\sigma = \frac{1}{2} (\mathbf{y} + \mathbf{y}^T)$ [S1].
4.  **Compute Von Mises Stress:** Calculate $\rho$ using the formula above with components of $\sigma$ [S1].
5.  **Map:** Apply a transfer function (e.g., logarithmic scaling and colormap) to $\rho$ to determine color and opacity [S2, unofficial].
6.  **Composite:** Integrate along the ray using standard volume rendering techniques (e.g., midpoint method) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_von_mises_stress_at_point(x, geometry_jacobian_func, displacement_jacobian_func):
    """
    Computes Von Mises stress at a single point x.
    geometry_jacobian_func: Returns J_phi(p)
    displacement_jacobian_func: Returns J_u(p)
    """
    # 1. Get Jacobians
    J_phi = geometry_jacobian_func(x)
    J_u = displacement_jacobian_func(x)
    
    # 2. Solve for J_u_comp_phi_inv
    # System: [J_phi]^T * Y = [J_u]^T
    # Y is the matrix J_{u o phi^-1}
    # Note: In practice, this involves solving a linear system for each column of Y
    J_u_comp_phi_inv = solve_linear_system(transpose(J_phi), transpose(J_u))
    
    # 3. Compute Strain Tensor Sigma
    # Sigma = 0.5 * (Y + Y^T)
    Sigma = 0.5 * (J_u_comp_phi_inv + transpose(J_u_comp_phi_inv))
    
    # 4. Compute Von Mises Stress rho
    # Components: Sigma_11, Sigma_22, Sigma_33, Sigma_12, Sigma_23, Sigma_31
    s11 = Sigma[0, 0]
    s22 = Sigma[1, 1]
    s33 = Sigma[2, 2]
    s12 = Sigma[0, 1]
    s23 = Sigma[1, 2]
    s31 = Sigma[2, 0]
    
    rho = (
        (s11 - s22)**2 + 
        (s22 - s33)**2 + 
        (s33 - s11)**2 + 
        6 * (s12**2 + s23**2 + s31**2)
    ) / 2.0
    
    return rho

def render_stress_volume(ray_origin, ray_direction, displacement_field, geometry):
    """
    Renders the volume using on-the-fly stress calculation.
    """
    # Standard volume rendering loop
    t = 0
    color = (0, 0, 0)
    alpha = 0
    
    while t < max_distance:
        x = ray_origin + t * ray_direction
        
        # Calculate stress on the fly
        rho = compute_von_mises_stress_at_point(x, geometry.jacobian, displacement_field.jacobian)
        
        # Apply transfer function (e.g., log scaling)
        # Note: Specific TF details depend on implementation [S2, unofficial]
        mapped_value = log10(rho + epsilon)
        rgba = transfer_function(mapped_value)
        
        # Composite
        color = color + rgba[3] * rgba[0] * (1 - alpha)
        alpha = alpha + rgba[3] * (1 - alpha)
        
        t += step_size
        
        if alpha > 0.99:
            break
            
    return color
```

**Parameters:**
*   **Sample Distance:** Dictated by the volume render integral; methods like RK2 (midpoint) use a fixed sample distance [S1].
*   **Colormap:** Dark-red-to-yellow is used in specific physiological mesh pipelines [S2, unofficial].
*   **Scaling:** Logarithmic scaling ($\log_{10}$) is applied to stress values in mesh-based pipelines [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Degeneracies:** Boundary conditions may introduce degeneracies in the Jacobian or displacement field, requiring robust solvers [S1].
*   **Point vs. Cell Data:** In mesh-based pipelines, stress must be computed as point-associated data to ensure correct averaging during smoothing; using cell-associated data leads to errors [S2, unofficial].
*   **Mesh Inversion:** Smoothing operations applied to the full tetrahedral volume can invert tetrahedra. Filtering must be restricted to the rendered surface geometry to prevent topological failure [S2, unofficial].

## Subtopics

### Stress Visualization and Smoothing Pipeline
A specialized pipeline for physiological stress data on tetrahedral meshes [S2, unofficial]. It differs from standard volume rendering by explicitly handling mesh topology for smoothing. It computes Von Mises stress from point-associated tensors, applies $\log_{10}$ scaling, maps to a dark-red-to-yellow colormap, smooths nodal stress across mesh edges, and applies a median filter exclusively to rendered surface positions to prevent tetrahedron inversion [S2, unofficial]. A key implementation detail is deduplicating bidirectional edge pairs in `_cell_edges` to ensure each distinct directed neighbor is averaged exactly once during the nodal smoothing pass [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** High per-sample cost due to the on-the-fly solution of linear systems for the strain tensor [S1]. The second-order midpoint method is noted as the fastest among tested ODE-based methods for the integration step [S1].
*   **Memory:** Low storage overhead for stress data as it is computed on the fly, but requires efficient access to displacement field control points and basis functions [S1].
*   **Performance:** In isogeometric analysis benchmarks (e.g., TERRIFIC project twisted bar), RK2 (midpoint) achieved ~78 ms total render time for a specific model, outperforming higher-order Runge-Kutta methods [S1].
*   **Strengths:** Pixel-accurate geometry representation; handles complex deformations defined by B-splines; avoids pre-computation storage costs [S1].
*   **Limitations:** Computationally expensive per sample; sensitive to degeneracies at boundaries [S1].

## Background
Originating from the field of isogeometric analysis, this method was detailed in work by the TERRIFIC project researchers [S1]. It addresses the need for visualizing structural analysis results where external forces lead to displacement fields $u: P \rightarrow \mathbb{R}^3$ [S1]. The approach was developed to allow interactive visualization of stress in complex geometries (e.g., twisted bars, backstep flows) without pre-computing stress fields on a dense grid [S1]. Later adaptations extended these concepts to physiological tetrahedral meshes, incorporating smoothing and surface filtering techniques [S2, unofficial].

## Sources
[S1] Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry, https://doi.org/10.1109/tvcg.2015.2430337, fetched 2026-10-28
[S2] ENH: Write UsdGeomTetMesh from tetrahedral grids; stress-colored tutorial 18, https://github.com/Project-MONAI/monai-physio/pull/152, fetched 2026-10-27 (unofficial)
