# Object Space Sampling via Gradient Integration and Directional Newton-Raphson

Also called: Initial Isosurface Sampling, iHOS Phase 1

## Overview
This method constitutes the first phase of the Interactive Isocontouring of High-Order Surfaces (iHOS) pipeline, designed to generate an initial set of points on an isosurface defined by high-order multivariate polynomials. It solves the problem of finding accurate root locations for polynomial fields where standard Newton-Raphson methods fail due to poor initial guesses. It sets itself apart from pure image-order ray casting by performing root finding in object space first, using gradient integration to guide seeds to the surface, thereby ensuring the subsequent directional Newton-Raphson steps converge reliably.

## How it works
The method operates in two distinct stages: object-space seed projection and image-space refinement. All computations are performed independently for each seed point, requiring no neighborhood information.

### Inputs and Outputs
*   **Inputs**: A high-order multivariate polynomial function $f(\mathbf{x})$ defining the implicit surface, a target isovalue (typically 0), and a uniform grid of cells covering the domain.
*   **Outputs**: A set of surface-tangent quads covering the isosurface in object space, which serve as rasterization targets for the next phase.

### Data Structures
*   **Seed Point**: A 3D vector $\mathbf{x}$ in reference space.
*   **Quad**: Defined by a center point (the projected seed) and a tangent plane basis, generated after projection.

### Step 1: Uniform Seed Distribution
Seeds are distributed uniformly within each cell of the domain grid. Let $N$ be the number of seeds per cell. The position $\mathbf{x}_0$ of a seed is chosen randomly or deterministically within the cell boundaries.

### Step 2: Gradient Integration
To mitigate the failure of Newton-Raphson when starting points are far from the solution, seeds are guided along the gradient field towards the isosurface. This integration step brings the seeds closer to the zero-crossing.
*   **Process**: The seed position is updated by integrating along the gradient direction.
*   **Goal**: Ensure the starting point for the Newton-Raphson step is within the basin of attraction for the desired root.

### Step 3: Directional Newton-Raphson Projection
Once seeds are guided close to the isosurface, Directional Newton-Raphson (NR) is applied to refine the projection onto the surface. This method is chosen over standard NR because it exhibits quadratic convergence [S1, book].

**Formula**:
The iteration updates the point position $\mathbf{x}$ as follows:

$$
\mathbf{x}_{k+1} = \mathbf{x}_k - \frac{f(\mathbf{x}_k)}{\nabla f(\mathbf{x}_k) \cdot \nabla f(\mathbf{x}_k)} \nabla f(\mathbf{x}_k)
$$

Where:
*   $k$ is the iteration index ($k = 0, 1, \dots$).
*   $\mathbf{x}_k$ is the point position in reference space at step $k$.
*   $f(\mathbf{x}_k)$ is the value of the polynomial for the current cell at $\mathbf{x}_k$.
*   $\nabla f(\mathbf{x}_k)$ is the gradient of the polynomial at $\mathbf{x}_k$.
*   $\nabla f(\mathbf{x}_k) \cdot \nabla f(\mathbf{x}_k)$ is the dot product of the gradient with itself (squared magnitude).

**Pitfall**: Standard Newton-Raphson may fail if uniformly distributed seeds are not close to the solution. The preceding gradient integration step is critical to mitigate this [S1, book].

### Step 4: Quad Generation
For each successfully projected point, a surface-tangent quad is generated. These quads are centered at the projected seed and aligned with the surface normal (gradient) at that point. These quads cover the isosurface and serve as the input for the image-space refinement phase.

### Step 5: Image Space Refinement (Ray Casting)
The quads generated in Step 4 are rasterized in image space. Fragments generated during rasterization serve as starting points for ray casting to further refine the isosurface representation.

**Process**:
1.  **Rasterization**: Rasterize the surface-tangent quads in image space to obtain fragment coordinates.
2.  **Transformation**: Transform the ray direction and fragment coordinates into the reference coordinate system.
3.  **Ray-Isosurface Intersection**: Perform ray-isosurface intersection computations using directional NR along the ray direction.

**Formula for Ray Casting**:
The directional NR iteration for fragment projection along a ray vector $\mathbf{r}$ is:

$$
\mathbf{x}_{k+1} = \mathbf{x}_k - \frac{f(\mathbf{x}_k)}{\nabla f(\mathbf{x}_k) \cdot \mathbf{r}} \mathbf{r}
$$

Where:
*   $\mathbf{x}_k$ is the point position in reference space.
*   $f(\mathbf{x}_k)$ is the polynomial value.
*   $\nabla f(\mathbf{x}_k)$ is the gradient.
*   $\mathbf{r}$ is the ray vector in reference space.

**Termination and Validation**:
*   If the maximum number of iterations is reached and the fragment is not projected onto the isosurface within an error tolerance bound, or if the point has moved outside the cell, the fragment is discarded.
*   If successfully projected, shading is computed and the corresponding pixel color is updated.

**Pitfall**: Fragments are processed independently, making it impossible to detect and directly fix holes in the isosurface during this phase. Holes can result from insufficient iteration counts, high curvature relative to initial sampling, or insufficient quad sizes [S1, book].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def object_space_sampling(polynomial_f, gradient_f, domain_cells, seeds_per_cell):
    """
    Phase 1: Object Space Sampling via Gradient Integration and Directional Newton-Raphson.
    Returns a list of surface-tangent quads.
    """
    quads = []
    
    for cell in domain_cells:
        # Step 1: Distribute seeds uniformly into each cell
        seeds = generate_uniform_seeds(cell, seeds_per_cell)
        
        for seed in seeds:
            x = seed
            
            # Step 2: Guide seeds through integration along the gradient field
            # Note: Specific integration step size and method (e.g., Euler, RK4) 
            # are not detailed in the source, but the goal is to get closer to the isosurface.
            x = integrate_gradient(x, gradient_f, polynomial_f)
            
            # Step 3: Project seeds using directional NR on the isosurface
            # Standard NR may fail if seeds are not close; integration mitigates this.
            x = directional_newton_raphson_gradient(x, polynomial_f, gradient_f)
            
            # Check if projection was successful (within tolerance and inside cell)
            if is_valid_projection(x, cell, polynomial_f):
                # Step 4: Generate surface-tangent quads for each projected point
                normal = gradient_f(x)
                quad = create_surface_tangent_quad(center=x, normal=normal)
                quads.append(quad)
                
    return quads

def directional_newton_raphson_gradient(x, f, grad_f, max_iter=10, tolerance=1e-6):
    """
    Directional NR along the gradient direction.
    Formula: x_{k+1} = x_k - (f(x_k) / (grad_f(x_k) . grad_f(x_k))) * grad_f(x_k)
    """
    for k in range(max_iter):
        fx = f(x)
        if abs(fx) < tolerance:
            break
            
        g = grad_f(x)
        g_dot_g = dot(g, g)
        
        if g_dot_g == 0:
            break # Gradient is zero, cannot proceed
            
        step = (fx / g_dot_g) * g
        x = x - step
        
    return x

def image_space_refinement(quads, ray_directions, max_iter=10, tolerance=1e-6):
    """
    Phase 2: Image Space Refinement via Rasterization and Ray-Casting.
    """
    # Step 1: Rasterize quads in image space to obtain fragment coordinates
    fragments = rasterize_quads(quads)
    
    results = []
    
    for frag in fragments:
        # Step 2: Transform ray and fragment coordinates into reference coordinate system
        x = transform_to_reference(frag.position)
        r = transform_to_reference(frag.ray_direction)
        
        # Step 3: Perform ray-isosurface intersection using directional NR
        # Formula: x_{k+1} = x_k - (f(x_k) / (grad_f(x_k) . r)) * r
        for k in range(max_iter):
            fx = f(x)
            if abs(fx) < tolerance:
                break
                
            g = grad_f(x)
            g_dot_r = dot(g, r)
            
            if g_dot_r == 0:
                break # Ray is parallel to gradient, cannot project along ray
                
            step = (fx / g_dot_r) * r
            x = x - step
            
            # Check if moved outside cell
            if not is_inside_cell(x):
                break
                
        # Step 4 & 5: Validation and Shading
        if is_valid_projection(x, current_cell, f) and abs(f(x)) < tolerance:
            color = compute_shading(x, grad_f(x))
            results.append((frag.pixel_id, color))
        else:
            # Discard fragment
            pass
            
    return results
```

## Subtopics

### Image Space Refinement via Rasterization and Ray-Casting
This is the second phase of the iHOS method. It takes the surface-tangent quads generated by the object space sampling and rasterizes them in image space. Fragments from this rasterization serve as starting points for ray casting. Directional Newton-Raphson is applied along the ray direction to refine the isosurface representation. It is chosen when high-quality, pixel-accurate rendering is required after the initial coarse sampling. A key limitation is that fragments are processed independently, making it impossible to detect and directly fix holes in the isosurface during this phase [S1, book].

## Operating characteristics
*   **Time Complexity**: The method performs independent computations for each seed and fragment. The complexity is linear with respect to the number of seeds and fragments, multiplied by the cost of polynomial evaluation and gradient computation per iteration.
*   **Memory Complexity**: Low, as no neighborhood information is needed. Data structures are limited to the seed points and the resulting quads.
*   **Strengths**: The gradient integration step significantly improves the robustness of Newton-Raphson convergence by ensuring starting points are close to the isosurface. The use of directional NR provides quadratic convergence.
*   **Limitations**: Holes in the isosurface can occur due to insufficient iteration counts, high curvature, or insufficient quad sizes. These holes cannot be fixed in the image-space phase because fragments are processed independently.
*   **When to use**: Suitable for interactive rendering of high-order polynomial isosurfaces where robustness of root finding is critical.
*   **When not to use**: Not suitable if hole-free surfaces are required without post-processing, or if the isosurface has extremely high curvature relative to the initial sampling density.

## Background
The method was introduced by C. Pagot, J. Vollrath, F. Sadlo, D. Weiskopf, T. Ertl, and J.L.D. Comba in the paper "Interactive Isocontouring of High-Order Surfaces" (2011). It is part of a GPU-based approach for interactive rendering of isosurfaces defined by high-order multivariate polynomials. The method builds on the principle that standard Newton-Raphson methods require good initial guesses, which are provided by gradient integration in object space.

## Sources
[S1] Interactive Isocontouring of High-Order Surfaces, https://doi.org/10.4230/dfu.vol2.sciviz.2011.276, fetched 2026-10-29 (book)
