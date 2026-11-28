# Parabolic Sphere Tracing

Also called: Analytical tracing in a single cell, Parabolic ray tracing

## Overview
Parabolic Sphere Tracing is a ray tracing method where the ray trajectory is a second-order polynomial (parabola) because the squared refractive index (RIF) is defined as a linear function of position, resulting in a constant gradient of squared RIF [S1]. It solves the ray tracing equations analytically within a single tetrahedral cell, allowing for exact intersection tests with planar boundaries [S1]. It sets itself apart from standard sphere tracing by replacing iterative distance-field stepping with closed-form polynomial evaluation, and from [Non-Linear Sphere Tracing](4079-non-linear-sphere-tracing.md) by avoiding numerical integration (Runge-Kutta) in favor of exact algebraic solutions when the RIF gradient is constant [G].

## How it works
**Inputs:**
- Initial ray origin $\mathbf{x}_0$ and momentum $\mathbf{p}_0$ [S1].
- A voxel grid defining the medium, where each voxel is subdivided into tetrahedral cells [S1].
- Squared RIF values $\eta^2_i$ at the vertices of the current tetrahedral cell [S1].
- A Signed Distance Field (SDF) $S(\mathbf{x})$ for the medium boundaries (e.g., glass panels, bubbles) [S1].

**Outputs:**
- Intersection point $\mathbf{x}_{hit}$ and parameter $\tau_{hit}$ along the parabolic trajectory [S1].
- Updated ray state $(\mathbf{x}_{next}, \mathbf{p}_{next})$ for the next cell [S1].

**Data Structures:**
- **RIF Grid:** A 3D grid of voxels. Each voxel contains 4 vertices with associated squared RIF values $\eta^2$ [S1]. The grid resolution must be a multiple of the bubble voxel grid resolution to guarantee no intersection with a bubble is missed while stepping through the tetrahedron [S1].
- **Tetrahedral Cell:** Defined by 4 vertices $\mathbf{v}_0, \dots, \mathbf{v}_3$ [S1]. The RIF within the cell is interpolated using barycentric coordinates [S1].

**Step-by-Step Method:**

1.  **Define Local RIF Field:**
    Within the current tetrahedral cell, define the squared refractive index $\eta^2(\mathbf{x})$ as a linear function of position:
    $$ \eta^2(\mathbf{x}) = A_0 + \boldsymbol{\gamma} \cdot \mathbf{x} $$
    where $\boldsymbol{\gamma} = [A_1, A_2, A_3]^T$ is the constant gradient of the squared RIF, and $A_0$ is the offset [S1]. The coefficients $\{A_0, A_1, A_2, A_3\}$ are derived from the barycentric interpolation of the squared RIF values $\{\eta^2_0, \eta^2_1, \eta^2_2, \eta^2_3\}$ at the cell's vertices [S1].

2.  **Initialize Ray State:**
    Set the initial ray parameter $\tau = 0$ [S1]. The initial position is $\mathbf{x}_0$ and initial momentum is $\mathbf{p}_0$ [S1]. The condition $\|\mathbf{p}(\tau)\| = \eta(\mathbf{x}(\tau))$ must hold at $\tau=0$ [S1].

3.  **Compute Parabolic Trajectory:**
    The ray position and momentum are given by second-order polynomials:
    $$ \mathbf{x}(\tau) = \frac{1}{4}\boldsymbol{\gamma} \tau^2 + \mathbf{p}_0\tau + \mathbf{x}_0 $$
    $$ \mathbf{p}(\tau) = \frac{1}{2}\boldsymbol{\gamma} \tau + \mathbf{p}_0 $$
    The condition $\|\mathbf{p}(\tau)\|^2 = \eta^2(\mathbf{x}(\tau))$ is satisfied for all $\tau$ if it holds at $\tau=0$ [S1].

4.  **Determine Safe Step Size $\Delta \tau$:**
    To avoid missing intersections with SDF boundaries, compute a safe upper bound for the step in parameter space [S1]. The arc length $s$ is related to $\tau$ by $ds = \eta(\mathbf{x}(\tau)) d\tau$ [S1]. Since $\eta \ge \min(\eta_{cell})$ within the current cell, a safe step $\Delta \tau$ corresponding to a physical distance $S(\mathbf{x}_k)$ is:
    $$ \Delta \tau = \frac{S(\mathbf{x}_k)}{\min(\eta_{cell})} $$
    Using $\Delta \tau = S(\mathbf{x}_k)$ directly is incorrect because $\eta \ge 1$ implies physical distance is larger than the parameter step [S1].

5.  **Find Intersection:**
    Solve for the smallest positive $\tau$ where the ray intersects the SDF boundary or the cell boundary [S1].
    -   **Planar Cell Boundary:** If the cell boundary is a plane defined by $\mathbf{n} \cdot \mathbf{x} = q$, solve the quadratic equation for $\tau$:
        $$ \frac{1}{4}(\mathbf{n} \cdot \boldsymbol{\gamma}) \tau^2 + (\mathbf{n} \cdot \mathbf{p}_0) \tau + (\mathbf{n} \cdot \mathbf{x}_0 - q) = 0 $$
        [S1].
    -   **SDF Intersection:** Solve $\|\mathbf{x}(\tau) - \mathbf{x}_{sdf}\| = S(\mathbf{x}(\tau))$ numerically or analytically if the SDF is simple [G].

6.  **Update Ray State:**
    If an intersection is found at $\tau_{hit}$:
    -   Update position: $\mathbf{x}_{next} = \mathbf{x}(\tau_{hit})$ [S1].
    -   Update momentum: $\mathbf{p}_{next} = \mathbf{p}(\tau_{hit})$ [S1].
    -   If the intersection is with a medium boundary (e.g., glass surface), apply refraction/reflection laws using the surface normal and the new $\eta(\mathbf{x}_{next})$ [S1].
    -   If the intersection is with a cell boundary, transition to the adjacent cell, recompute $\boldsymbol{\gamma}$ and $A_0$ for the new cell, and repeat from Step 2 [S1].

**Pitfalls:**
- The ray parameter $\tau$ is not arc length $s$. Since $ds = \eta d\tau$ and $\eta \ge 1$, using $\Delta \tau = S(\mathbf{x}_k)$ may lead to missed intersections [S1].
- If the ray starts near an SDF discontinuity, $S(\mathbf{x}_k)$ may be overestimated. To avoid errors, the RIF grid must be larger than 1 in the glass thickness direction so that a ray inside a tetrahedron does not cover both the discontinuity zone and a panel face [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def parabolic_sphere_trace(x0, p0, cell_vertices, eta2_vertices, sdf_func, min_eta_cell):
    """
    Trace a ray through a single tetrahedral cell with linear squared RIF.
    
    Args:
        x0: Initial position (3,)
        p0: Initial momentum/direction (3,)
        cell_vertices: Array of 4 vertex positions (4, 3)
        eta2_vertices: Array of 4 squared RIF values at vertices (4,)
        sdf_func: Function returning SDF value and gradient at a point
        min_eta_cell: Minimum RIF in the cell for safe stepping
        
    Returns:
        hit_point: Intersection point (3,) or None
        hit_tau: Parameter tau at intersection
        next_p: Momentum after intersection (if boundary hit)
    """
    # 1. Define Local RIF Field [G]
    gamma, A0 = compute_rif_coefficients(cell_vertices, eta2_vertices)
    
    # 2. Initialize Ray State [S1]
    tau = 0.0
    x_curr = x0
    p_curr = p0
    # Ensure Eikonal equation ||p|| = eta holds at start [S1]
    eta_curr = np.sqrt(A0 + np.dot(gamma, x_curr))
    p_curr = p_curr / np.linalg.norm(p_curr) * eta_curr
    
    # 3. Determine Safe Step Size [S1]
    s_val, _ = sdf_func(x_curr)
    delta_tau = s_val / min_eta_cell
    
    # 4. Find Intersection [S1]
    hit_tau = None
    hit_point = None
    
    # Check planar cell boundaries
    for n, q in get_tetrahedron_faces(cell_vertices):
        a = 0.25 * np.dot(n, gamma)
        b = np.dot(n, p_curr)
        c = np.dot(n, x_curr) - q
        roots = solve_quadratic(a, b, c)
        for t in roots:
            if 0 < t <= delta_tau:
                if hit_tau is None or t < hit_tau:
                    hit_tau = t
                    hit_point = 0.25 * gamma * t**2 + p_curr * t + x_curr
                    
    # Check SDF intersection [G]
    # [G] The sources do not give the SDF intersection solver.
    # Implement a numerical root finder (e.g., bisection) for f(tau) = ||x(tau) - x_sdf||^2 - S(x(tau))^2 = 0.
    
    if hit_tau is not None:
        p_hit = 0.5 * gamma * hit_tau + p_curr
        return hit_point, hit_tau, p_hit
    return None, None, None

def compute_rif_coefficients(vertices, eta2_vals):
    # [G] Barycentric interpolation gradients for a tetrahedron
    v0, v1, v2, v3 = vertices
    vol = np.dot(v1 - v0, np.cross(v2 - v0, v3 - v0)) / 6.0
    g0 = np.cross(v1 - v3, v2 - v3) / (2.0 * vol)
    g1 = np.cross(v0 - v3, v2 - v3) / (2.0 * vol)
    g2 = np.cross(v0 - v3, v1 - v3) / (2.0 * vol)
    g3 = np.cross(v0 - v1, v0 - v2) / (2.0 * vol)
    gamma = eta2_vals[0] * g0 + eta2_vals[1] * g1 + eta2_vals[2] * g2 + eta2_vals[3] * g3
    A0 = eta2_vals[0] - np.dot(gamma, v0)
    return gamma, A0

def get_tetrahedron_faces(vertices):
    v0, v1, v2, v3 = vertices
    faces = [
        (np.cross(v1 - v0, v2 - v0), np.dot(np.cross(v1 - v0, v2 - v0), v0)),
        (np.cross(v0 - v1, v3 - v1), np.dot(np.cross(v0 - v1, v3 - v1), v1)),
        (np.cross(v0 - v2, v3 - v2), np.dot(np.cross(v0 - v2, v3 - v2), v2)),
        (np.cross(v1 - v2, v3 - v2), np.dot(np.cross(v1 - v2, v3 - v2), v2))
    ]
    return faces

def solve_quadratic(a, b, c):
    if abs(a) < 1e-12:
        if abs(b) < 1e-12:
            return []
        return [-c / b]
    disc = b**2 - 4*a*c
    if disc < 0:
        return []
    sqrt_disc = np.sqrt(disc)
    t1 = (-b + sqrt_disc) / (2*a)
    t2 = (-b - sqrt_disc) / (2*a)
    return [t1, t2]
```

## Subtopics

### [Parabolic Sphere Tracing for Heterogeneous Media Gradient Estimation](14538-parabolic-sphere-tracing-for-heterogeneous-media-gradient-es.md)
Estimates transmitted caustics by finding stationary optical paths through a heterogeneous refractive medium using a gradient-based optimizer. It sets itself apart by explicitly deriving gradient corrections for spatially varying RIF, enabling quasi-Newton solvers.

### [Paraxial Ray Tracing](14547-paraxial-ray-tracing.md)
Traces rays in the vicinity of a central ray by computing first-order perturbations. It builds on Parabolic Sphere Tracing by leveraging its analytical properties to efficiently propagate ray bundles for caustic estimation.

## Operating characteristics
- **Time Complexity:** $O(1)$ per cell for trajectory evaluation and intersection, assuming constant-time SDF evaluation [G]. This is faster than iterative sphere tracing which requires multiple distance queries per step [G].
- **Memory Complexity:** $O(1)$ per cell. Requires storing vertex RIF values and SDF data [G].
- **Measured Performance:** In the source, rendering was done at 1024x1024 resolution with 1M samples per pixel [S1]. The RIF grid resolution was 1024x1024x2, and the bubble voxel grid was 64x64x1 [S1].
- **Strengths:** Exact intersection with planar boundaries; no numerical integration errors; efficient for media with piecewise linear squared RIF [S1].
- **Limitations:** Requires the squared RIF to be linear within cells; step size must be carefully bounded to avoid missing SDF intersections; not suitable for arbitrary non-linear RIF fields without subdivision [S1].
- **When to use:** Rendering hand-blown glass panels with bubbles and surface irregularities where the RIF can be approximated as linear within small tetrahedral cells [S1].
- **When not to use:** When the RIF varies non-linearly within the cell and subdivision is too expensive, or when high-order accuracy is required without cell subdivision [G].

## Background
Parabolic Sphere Tracing was introduced in the paper "Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering" [S1]. It is used for rendering heterogeneous media such as old glass panels, handling surface irregularities, bubbles, and complex transmitted caustics [S1]. The method leverages the analytical properties of parabolic trajectories to simplify ray tracing in media with linearly varying squared refractive index [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
