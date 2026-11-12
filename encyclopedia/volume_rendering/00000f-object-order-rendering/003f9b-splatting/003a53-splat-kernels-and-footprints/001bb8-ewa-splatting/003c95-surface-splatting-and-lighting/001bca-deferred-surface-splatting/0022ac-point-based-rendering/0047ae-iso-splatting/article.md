# Iso-splatting

Also called: Gradient-based Point Splatting, Point-based Isosurface Splatting

## Overview
Iso-splatting is a point-based alternative to isosurface visualization that generates point samples per cell instead of triangles, projecting them onto the isosurface using function values and gradients [S1]. It splits extraction into two independent steps: generating point samples throughout the volumetric domain, and projecting these points onto the isosurface of interest, followed by rendering using a surface splatting algorithm [S1]. Unlike standard polygon-based methods where geometry is discarded and recomputed for each new isovalue, iso-splatting uses point geometry that can represent a range of isosurfaces, requiring only small adjustments for slightly different isovalue, thereby decreasing computation time and storage [S1]. It builds on [Point-Based Rendering](2827-point-based-rendering.md) by utilizing surface splatting kernels to render the projected points.

## How it works
**Inputs:** A scalar field defined on a grid (or point cloud), an isovalue $f_I$, and a displacement threshold $\Delta d$.
**Outputs:** A set of projected point samples with associated normals and variance matrices, ready for surface splatting.

**Data Structures:**
Each point sample in the intermediate representation must store:
1.  **Location:** A 3D vector $\mathbf{p}$ (or $\mathbf{x}$) representing the sample position in the volumetric domain [S1].
2.  **Function Value:** A scalar float $f(\mathbf{p})$ [S1].
3.  **Gradient:** A 3D vector $\nabla f(\mathbf{p})$ [S1].
4.  **Local Variance Matrix:** A $3 \times 3$ matrix (or derived parameters) indicating the average distance to neighboring samples, used for EWA splatting [S1]. For uniform rectilinear grids, this can be derived from grid spacing [S1].

**Algorithm Steps:**

1.  **Generate Point Samples:**
    Generate point samples $\mathbf{p}_\lambda$ throughout the volumetric domain of the scalar function. For grid data, this typically involves placing one sample per cell or per active cell [S1]. If the grid is sparse in one dimension, cells may be subdivided until the function value interval width of child cells is below a threshold [S1].

2.  **Compute Displacement Threshold:**
    Calculate the maximum allowed displacement $\Delta d$ based on the dimensions of the cell containing the sample. Given cell dimensions $s_x, s_y, s_z$:
    $$ \Delta d = \frac{1}{2} \sqrt{s_x^2 + s_y^2 + s_z^2} $$
    Geometrically, this restricts the projected point to reside inside a sphere of radius $\Delta d$ centered at the initial guess $\mathbf{p}_\lambda$ [S1].

3.  **Evaluate Function and Gradient:**
    At each sample point $\mathbf{p}_\lambda$ (denoted as $\mathbf{x}_0$ in the projection formula), evaluate the function value $f(\mathbf{p}_\lambda)$ and the gradient $\nabla f(\mathbf{p}_\lambda)$. For grid data, this is done efficiently using trilinear interpolation [S1].

4.  **Project to Isosurface (Newton-Raphson Approximation):**
    Compute the displacement parameter $t$ using the Newton-Raphson approximation for a single iteration:
    $$ t = \frac{f_I - f(\mathbf{p}_\lambda)}{\nabla f(\mathbf{p}_\lambda) \cdot \nabla f(\mathbf{p}_\lambda)} $$
    where $f_I$ is the isovalue, $f(\mathbf{p}_\lambda)$ is the function value at the initial guess, and $\nabla f(\mathbf{p}_\lambda)$ is the gradient at the initial guess [S1].
    
    Compute the projected point $\mathbf{p}'_\lambda$:
    $$ \mathbf{p}'_\lambda = \mathbf{p}_\lambda + \nabla f(\mathbf{p}_\lambda) t $$
    [S1]

5.  **Validate Projection:**
    Check if the projected point is too far from the initial guess. Calculate the distance $d = \| \mathbf{p}'_\lambda - \mathbf{p}_\lambda \|$.
    - If $d > \Delta d$, discard the point. This handles cases where the Newton-Raphson method converges to roots far from the desired root, such as near local maxima or minima [S1].
    - If $d \le \Delta d$, keep the point. The normal for splatting can be derived from $\nabla f(\mathbf{p}'_\lambda)$ (requiring a re-evaluation) or approximated by $\nabla f(\mathbf{p}_\lambda)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iso_splatting(scalar_field, isovalue, grid_spacing):
    """
    scalar_field: Function or data structure allowing evaluation of f(p) and grad_f(p)
    isovalue: float f_I
    grid_spacing: tuple (sx, sy, sz) defining cell dimensions
    """
    sx, sy, sz = grid_spacing
    # Step 2: Compute displacement threshold
    delta_d = 0.5 * (sx**2 + sy**2 + sz**2)**0.5
    
    projected_points = []
    
    # Step 1: Generate point samples (e.g., one per cell center)
    # In a real implementation, this iterates over active cells or grid indices
    for p_lambda in generate_samples(scalar_field):
        
        # Step 3: Evaluate function and gradient
        f_val = scalar_field.evaluate(p_lambda)
        grad_val = scalar_field.gradient(p_lambda)
        
        # Step 4: Project to isosurface
        # t = (f_I - f(p)) / (grad(p) . grad(p))
        grad_dot = dot(grad_val, grad_val)
        
        # Avoid division by zero if gradient is zero (local extremum)
        if grad_dot == 0:
            continue
            
        t = (isovalue - f_val) / grad_dot
        
        # p_prime = p + grad * t
        p_prime = p_lambda + grad_val * t
        
        # Step 5: Validate projection
        displacement = norm(p_prime - p_lambda)
        
        if displacement <= delta_d:
            # Store projected point and necessary info for splatting
            # Normal can be computed at p_prime or approximated by grad_val
            projected_points.append({
                'location': p_prime,
                'gradient': grad_val, # Or re-evaluate at p_prime for accuracy
                'variance_matrix': compute_variance_matrix(grid_spacing)
            })
            
    return projected_points
```

**Parameters:**
-   **Displacement threshold $\Delta d$:** Computed from cell dimensions. Typical value is half the diagonal of the cell [S1].
-   **Cell subdivision criterion:** Function value interval width. Cells spanning large value ranges are subdivided until child interval width < threshold [S1].

**Edge Cases and Pitfalls:**
-   **Local Extrema:** Near local maxima or minima, the gradient is small, causing $t$ to become large and the Newton-Raphson step to overshoot. The displacement criterion $\Delta d$ discards these "far-away" solutions [S1].
-   **Multiple Roots:** A single cell may contain several disjoint components of the same isosurface. Iso-splatting generates only one point per cell, potentially missing fine structures or multiple components [S1].
-   **Sparse Sampling:** If the grid is sparse in one dimension, rendering quality may degrade. Subdivision or altering the variance matrix can compensate [S1].

## Subtopics

### Preprocessed Iso-splatting
Iso-splatting paradigm where point generation and projection are both performed in the preprocessing stage [S1]. Storage is high, requiring storing location, function value, gradient, and function value interval for each point sample [S1]. It uses interval trees for active-cell lookup [S1]. It is more computation-intensive than paradigms where projection is deferred, but offers better storage efficiency than paradigms where only generation is preprocessed [S1].

### Deferred Iso-splatting
Iso-splatting paradigm where point generation is in preprocessing, but projection is performed in the rendering stage [S1]. Alternatively, point generation is in extraction, but projection is performed in the rendering stage [S1]. Hardware requirements include support for branches for the displacement criterion or texture lookups for discarded points [S1]. This allows interactive adjustment of the isovalue without recomputing the projection for all points [S1].

## Operating characteristics
**Time Complexity:**
Iso-splatting (Paradigm C, where generation and projection are in extraction) takes approximately half the time of standard Marching Cubes (MC) on a Pentium 4 PC with GeForce 4 Ti [S1]. The extraction involves a single function value and gradient computation (simultaneous via trilinear interpolation) followed by one iteration of Newton-Raphson, which is cheaper than MC's edge intersections and triangulation [S1].

**Memory Complexity:**
Iso-splatting generates one point per cell ($n$ vertices), while triangle-soup MC generates $3n$ vertices (assuming one triangle per cell) [S1]. Standard MC implementations often generate more than one triangle per cell, making iso-splatting significantly more storage-efficient [S1]. However, paradigms (A) and (B) where points are pre-generated are storage-intensive [S1].

**Measured Performance:**
-   **Framerate:** Iso-splatting with quadrilateral splats provides quadruple the framerate of a standard MC implementation on the same hardware [S1].
-   **Hardware:** Implemented with QSplat-style surface splatting, using quadrilateral and elliptical surface splatting kernels [S1]. Quadrilateral splats are efficient as they can be implemented by passing a single vertex to graphics hardware [S1].

**Strengths:**
-   High interactivity due to fast extraction and rendering [S1].
-   Storage efficiency compared to triangle-based methods [S1].
-   Flexibility to represent a range of isosurfaces with minimal adjustment [S1].
-   Can be extended to out-of-core or parallel environments [S1].

**Limitations:**
-   One point per cell may miss fine details or multiple components within a cell [S1].
-   Requires fairly regular sampling of the surface for proper rendering [S1].
-   Approximate projection may introduce errors if the displacement threshold is too loose or the field is highly non-linear [S1].

**When to Use:**
Use when high interactivity and storage efficiency are prioritized over geometric completeness, and when the scalar field is sufficiently dense [S1].

**When Not to Use:**
Avoid when the isosurface has complex topology with multiple components per cell or when exact geometric representation is required [S1].

## Background
Iso-splatting was introduced by C. Olofsson, U. Hamann, and K.-L. Joy in 2003 [S1]. The method was developed to address the computational and storage costs of standard triangle-based isosurfacing methods like Marching Cubes [S1]. It leverages the efficiency of point-based rendering and surface splatting techniques (such as QSplat and EWA splatting) to visualize isosurfaces of scalar fields [S1]. The approach allows for the distribution of work across preprocessing, extraction, and rendering stages, enabling optimization for specific application needs such as out-of-core processing or parallel execution [S1].

## Sources
[S1] Iso-splatting: a point-based alternative to isosurface visualization, https://doi.org/10.1109/pccga.2003.1238274, fetched 2026-10-26
