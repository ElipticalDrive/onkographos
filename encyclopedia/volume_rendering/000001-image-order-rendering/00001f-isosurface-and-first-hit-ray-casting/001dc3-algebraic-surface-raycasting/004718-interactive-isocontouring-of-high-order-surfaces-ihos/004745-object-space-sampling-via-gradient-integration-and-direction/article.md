# Object Space Sampling via Gradient Integration and Directional Newton-Raphson

Also called: Initial Isosurface Sampling, iHOS Phase 1

## Overview
This method constitutes a two-step process to find isosurface points for polynomial data [S1, book]. It solves the problem of finding accurate root locations for polynomial fields where standard Newton-Raphson methods fail due to poor initial guesses [S1, book]. It sets itself apart from pure image-order ray casting by performing root finding in object space first, using gradient integration to guide seeds to the surface, thereby ensuring the subsequent directional Newton-Raphson steps converge reliably [S1, book].

## How it works
The method operates in two distinct stages: object-space seed projection and image-space refinement [S1, book]. All computations are performed independently for each seed point, requiring no neighborhood information [S1, book].

### Inputs and Outputs
*   **Inputs**: A high-order multivariate polynomial function $f(\mathbf{x})$ defining the implicit surface, a target isovalue (typically 0), and a uniform grid of cells covering the domain [S1, book].
*   **Outputs**: A set of surface-tangent quads covering the isosurface in object space, which serve as rasterization targets for the next phase [S1, book].

### Data Structures
*   **Seed Point**: A 3D vector $\mathbf{x}$ in reference space [S1, book].
*   **Quad**: Defined by a center point (the projected seed) and a tangent plane basis, generated after projection [S1, book].

### Step 1: Uniform Seed Distribution
Seeds are distributed uniformly within each cell of the domain grid [S1, book]. The position $\mathbf{x}_0$ of a seed is chosen randomly or deterministically within the cell boundaries [G].

### Step 2: Gradient Integration
To mitigate the failure of Newton-Raphson when starting points are far from the solution, seeds are guided along the gradient field towards the isosurface [S1, book]. This integration step brings the seeds closer to the zero-crossing [S1, book].

### Step 3: Directional Newton-Raphson Projection
Once seeds are guided close to the isosurface, Directional Newton-Raphson (NR) is applied to refine the projection onto the surface [S1, book]. This method is chosen over standard NR because it exhibits quadratic convergence [S1, book].

**Formula**:
The iteration updates the point position $\mathbf{x}$ as follows:

$$
\mathbf{x}_{k+1} = \mathbf{x}_k - \frac{f(\mathbf{x}_k)}{\nabla f(\mathbf{x}_k) \cdot \nabla f(\mathbf{x}_k)} \nabla f(\mathbf{x}_k)
$$

Where:
*   $k$ is the iteration index ($k = 0, 1, \dots$) [S1, book].
*   $\mathbf{x}_k$ is the point position in reference space at step $k$ [S1, book].
*   $f(\mathbf{x}_k)$ is the value of the polynomial for the current cell at $\mathbf{x}_k$ [S1, book].
*   $\nabla f(\mathbf{x}_k)$ is the gradient of the polynomial at $\mathbf{x}_k$ [S1, book].
*   $\nabla f(\mathbf{x}_k) \cdot \nabla f(\mathbf{x}_k)$ is the dot product of the gradient with itself (squared magnitude) [S1, book].

**Pitfall**: Standard Newton-Raphson may fail if uniformly distributed seeds are not close to the solution [S1, book]. The preceding gradient integration step is critical to mitigate this [S1, book].

### Step 4: Quad Generation
For each successfully projected point, a surface-tangent quad is generated [S1, book]. These quads are centered at the projected seed and aligned with the surface normal (gradient) at that point [S1, book]. These quads cover the isosurface and serve as the input for the image-space refinement phase [S1, book].

### Step 5: Image Space Refinement (Ray Casting)
The quads generated in Step 4 are rasterized in image space [S1, book]. Fragments generated during rasterization serve as starting points for ray casting to further refine the isosurface representation [S1, book].

**Process**:
1.  **Rasterization**: Rasterize the surface-tangent quads in image space to obtain fragment coordinates [S1, book].
2.  **Transformation**: Transform the ray direction and fragment coordinates into the reference coordinate system [S1, book].
3.  **Ray-Isosurface Intersection**: Perform ray-isosurface intersection computations using directional NR along the ray direction [S1, book].

**Formula for Ray Casting**:
The directional NR iteration for fragment projection along a ray vector $\mathbf{r}$ is:

$$
\mathbf{x}_{k+1} = \mathbf{x}_k - \frac{f(\mathbf{x}_k)}{\nabla f(\mathbf{x}_k) \cdot \mathbf{r}} \mathbf{r}
$$

Where:
*   $\mathbf{x}_k$ is the point position in reference space [S1, book].
*   $f(\mathbf{x}_k)$ is the polynomial value [S1, book].
*   $\nabla f(\mathbf{x}_k)$ is the gradient [S1, book].
*   $\mathbf{r}$ is the ray vector in reference space [S1, book].

**Termination and Validation**:
*   If the maximum number of iterations is reached and the fragment is not projected onto the isosurface within an error tolerance bound, or if the point has moved outside the cell, the fragment is discarded [S1, book].
*   If successfully projected, shading is computed and the corresponding pixel color is updated [S1, book].

**Pitfall**: Fragments are processed independently, making it impossible to detect and directly fix holes in the isosurface during this phase [S1, book]. Holes can result from insufficient iteration counts, high curvature relative to initial sampling, or insufficient quad sizes [S1, book].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dot(a, b):
    return sum(ai * bi for ai, bi in zip(a, b))

def magnitude_sq(v):
    return dot(v, v)

def generate_uniform_seed_in_cell(cell):
    # Not given in the sources. Standard uniform distribution over cell bounds is assumed [G].
    return [cell.min[i] + (cell.max[i] - cell.min[i]) * 0.5 for i in range(len(cell.min))]

def integrate_gradient_step(x, grad_f, step_size):
    # Not given in the sources. Standard Euler integration is assumed [G].
    g = grad_f(x)
    return [xi + step_size * gi for xi, gi in zip(x, g)]

def is_inside_cell(x, cell):
    # Not given in the sources. Standard bounds check is assumed [G].
    return all(c_min <= xi <= c_max for xi, (c_min, c_max) in zip(x, zip(cell.min, cell.max)))

def is_valid_projection(x, cell, f, tolerance):
    # Not given in the sources. Standard bounds and value check is assumed [G].
    return abs(f(x)) < tolerance and is_inside_cell(x, cell)

def create_surface_tangent_quad(center, normal):
    # Not given in the sources. Standard tangent plane quad generation is assumed [G].
    return {"center": center, "normal": normal}

def rasterize_quads(quads):
    # Not given in the sources. Standard GPU rasterization is assumed [G].
    return [{"position": q["center"], "ray_direction": q["normal"]} for q in quads]

def transform_to_reference(coord):
    # Not given in the sources. Standard coordinate transformation is assumed [G].
    return list(coord)

def compute_shading(x, grad_f):
    # Not given in the sources. Standard shading computation is assumed [G].
    return [0.5, 0.5, 0.5]

def directional_newton_raphson_gradient(x, f, grad_f, max_iter, tolerance):
    for k in range(max_iter):
        fx = f(x)
        if abs(fx) < tolerance:
            break
        g = grad_f(x)
        g_dot_g = magnitude_sq(g)
        if g_dot_g == 0:
            break
        step = (fx / g_dot_g) * g
        x = [xi - s for xi, s in zip(x, step)]
    return x

def directional_newton_raphson_ray(x, r, f, grad_f, max_iter, tolerance):
    for k in range(max_iter):
        fx = f(x)
        if abs(fx) < tolerance:
            break
        g = grad_f(x)
        g_dot_r = dot(g, r)
        if g_dot_r == 0:
            break
        step = (fx / g_dot_r) * r
        x = [xi - s for xi, s in zip(x, step)]
    return x

def object_space_sampling(f, grad_f, domain_cells, seeds_per_cell, step_size, max_iter, tolerance):
    quads = []
    for cell in domain_cells:
        for _ in range(seeds_per_cell):
            x = generate_uniform_seed_in_cell(cell)
            x = integrate_gradient_step(x, grad_f, step_size)
            x = directional_newton_raphson_gradient(x, f, grad_f, max_iter, tolerance)
            if is_valid_projection(x, cell, f, tolerance):
                normal = grad_f(x)
                quad = create_surface_tangent_quad(center=x, normal=normal)
                quads.append(quad)
    return quads

def image_space_refinement(quads, cells, f, grad_f, max_iter, tolerance):
    fragments = rasterize_quads(quads)
    results = []
    for frag, cell in zip(fragments, cells):
        x = transform_to_reference(frag["position"])
        r = transform_to_reference(frag["ray_direction"])
        for k in range(max_iter):
            fx = f(x)
            if abs(fx) < tolerance:
                break
            g = grad_f(x)
            g_dot_r = dot(g, r)
            if g_dot_r == 0:
                break
            step = (fx / g_dot_r) * r
            x = [xi - s for xi, s in zip(x, step)]
            if not is_inside_cell(x, cell):
                break
        if is_valid_projection(x, cell, f, tolerance):
            color = compute_shading(x, grad_f(x))
            results.append((frag["pixel_id"], color))
        else:
            pass
    return results
```

## Subtopics

### Image Space Refinement via Rasterization and Ray-Casting
This is the second phase of the iHOS method [S1, book]. It takes the surface-tangent quads generated by the object space sampling and rasterizes them in image space [S1, book]. Fragments from this rasterization serve as starting points for ray casting [S1, book]. Directional Newton-Raphson is applied along the ray direction to refine the isosurface representation [S1, book]. It is chosen when high-quality, pixel-accurate rendering is required after the initial coarse sampling [S1, book]. A key limitation is that fragments are processed independently, making it impossible to detect and directly fix holes in the isosurface during this phase [S1, book].

## Operating characteristics
*   **Time Complexity**: The method performs independent computations for each seed and fragment [S1, book]. The complexity is linear with respect to the number of seeds and fragments, multiplied by the cost of polynomial evaluation and gradient computation per iteration [S1, book].
*   **Memory Complexity**: Low, as no neighborhood information is needed [S1, book]. Data structures are limited to the seed points and the resulting quads [S1, book].
*   **Strengths**: The gradient integration step significantly improves the robustness of Newton-Raphson convergence by ensuring starting points are close to the isosurface [S1, book]. The use of directional NR provides quadratic convergence [S1, book].
*   **Limitations**: Holes in the isosurface can occur due to insufficient iteration counts, high curvature, or insufficient quad sizes [S1, book]. These holes cannot be fixed in the image-space phase because fragments are processed independently [S1, book].
*   **When to use**: Suitable for interactive rendering of high-order polynomial isosurfaces where robustness of root finding is critical [S1, book].
*   **When not to use**: Not suitable if hole-free surfaces are required without post-processing, or if the isosurface has extremely high curvature relative to the initial sampling density [S1, book].

## Background
The method was introduced by C. Pagot, J. Vollrath, F. Sadlo, D. Weiskopf, T. Ertl, and J.L.D. Comba in the paper "Interactive Isocontouring of High-Order Surfaces" (2011) [S1, book]. It is part of a GPU-based approach for interactive rendering of isosurfaces defined by high-order multivariate polynomials [S1, book]. The method builds on the principle that standard Newton-Raphson methods require good initial guesses, which are provided by gradient integration in object space [S1, book].

## Sources
[S1] Interactive Isocontouring of High-Order Surfaces, https://doi.org/10.4230/dfu.vol2.sciviz.2011.276, fetched 2026-10-29 (book)
