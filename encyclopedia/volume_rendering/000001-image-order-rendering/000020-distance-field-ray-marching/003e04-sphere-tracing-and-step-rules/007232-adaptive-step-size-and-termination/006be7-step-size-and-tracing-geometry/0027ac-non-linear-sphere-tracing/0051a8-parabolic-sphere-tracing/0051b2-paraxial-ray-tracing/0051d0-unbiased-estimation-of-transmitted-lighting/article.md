# Unbiased Estimation of Transmitted Lighting

Also called: Fermat's Principle Path Tracing, Two-Point Tracing for Caustics

## Overview
This method estimates transmitted caustics by finding direct light paths between a receptor point and a light source using Fermat’s principle, handling both refractive interfaces and heterogeneous media [S1]. It distinguishes itself from surface-only techniques like Manifold Walk by characterizing curved light paths inside heterogeneous materials, allowing unified estimation of surface and volume-induced caustics [S1].

## How it works
**Inputs:**
- Receptor point $\mathbf{x}_0$.
- Light source point $\mathbf{x}_n$.
- $n-1$ intermediate points $\mathbf{x}_i$ ($1 \le i \le n-1$) lying on refractive interfaces defined by Signed Distance Functions (SDFs) $g_i$.
- Refractive Index Function (RIF) $\eta(\mathbf{x})$.

**Outputs:**
- The optical path length $L$ and the path configuration satisfying Fermat's principle.

**Data Structures:**
- The RIF $\eta(\mathbf{x})$ is often approximated within a single cell (e.g., a tetrahedron or voxel) such that $\eta^2(\mathbf{x})$ is linear:
  $$ \eta^2(\mathbf{x}) = A_0 + A_1 x + A_2 y + A_3 z = A_0 + \boldsymbol{\gamma} \cdot \mathbf{x} $$
  where $\boldsymbol{\gamma} = [A_1, A_2, A_3]^T$ is a constant gradient vector [S1].
- Bubbles are represented as opaque obstacles using a voxel grid [S1].

**Steps:**
1. **Formulate Optical Path Length:** Define the optical path length $L$ between $\mathbf{x}_0$ and $\mathbf{x}_n$ through the intermediate points:
   $$ L(\mathbf{x}_0, \dots, \mathbf{x}_n) = \sum_{k=0}^{n-1} \int_{\mathbf{x}_k}^{\mathbf{x}_{k+1}} \eta(\mathbf{x}) \, ds $$
   [S1]

2. **Apply Fermat's Principle:** Seek paths where the variation of the optical path length is zero ($dL = 0$). This defines a variational problem where the physical paths are the stationary points of $L$ [S1].

3. **Solve for Ray Trajectories in Heterogeneous Media:**
   If the RIF satisfies the linear squared condition $\eta^2(\mathbf{x}) = A_0 + \boldsymbol{\gamma} \cdot \mathbf{x}$, the ray trajectory $\mathbf{x}(\tau)$ and momentum $\mathbf{p}(\tau)$ are second-order polynomials [S1]:
   $$ \mathbf{x}(\tau) = \frac{1}{4}\boldsymbol{\gamma} \tau^2 + \mathbf{p}_0 \tau + \mathbf{x}_0 $$
   $$ \mathbf{p}(\tau) = \frac{1}{2}\boldsymbol{\gamma} \tau + \mathbf{p}_0 $$
   where $\mathbf{p}(\tau)$ is the momentum vector satisfying $\|\mathbf{p}(\tau)\| = \eta(\mathbf{x}(\tau))$ [S1]. The constant $\mathbf{p}_0$ is determined by the initial direction at $\mathbf{x}_0$ [S1].

4. **Handle Refractive Interfaces:**
   For interfaces defined by SDFs $g_i$, the intermediate points $\mathbf{x}_i$ must lie on the zero-level set of $g_i$. The solver adjusts $\mathbf{x}_i$ to minimize $L$ while satisfying $g_i(\mathbf{x}_i) = 0$ [S1].

5. **Check for Obstacles:**
   During the tracing of the path segments, check for intersections with bubbles. Bubbles are treated as opaque obstacles; if a path intersects a bubble, it is discarded or considered invalid [S1].

6. **Iterative Optimization:**
   Use a numerical optimizer to find the set of intermediate points $\{\mathbf{x}_1, \dots, \mathbf{x}_{n-1}\}$ that minimizes $L$ (or satisfies $dL=0$) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_fermat_path(x0, xn, interfaces, rif_params):
    """
    Solves for the path between x0 and xn using Fermat's principle.
    interfaces: list of SDF functions g_i
    rif_params: parameters for the RIF approximation (A0, gamma)
    """
    # Initialize intermediate points on interfaces
    # [G] Initial guess: project x0 and xn onto interfaces or use linear interpolation
    x_intermediate = [initial_guess_on_interface(i, x0, xn) for i in range(len(interfaces))]
    
    for iteration in range(max_iterations):
        # Calculate optical path length L
        L = 0.0
        points = [x0] + x_intermediate + [xn]
        
        for k in range(len(points) - 1):
            x_start = points[k]
            x_end = points[k+1]
            
            # If inside a heterogeneous cell with linear squared RIF:
            # Use analytical solution for integral of eta along the segment
            # [G] The integral of eta(s) for quadratic trajectory can be computed analytically
            # or via numerical quadrature if the quadratic assumption holds locally.
            # Here we assume the segment is short enough or the cell model applies.
            L += compute_optical_length(x_start, x_end, rif_params)
            
        # Compute gradient of L with respect to intermediate points
        grad_L = compute_gradient_L(L, x_intermediate, interfaces, rif_params)
        
        # Update intermediate points to minimize L
        # [G] Use gradient descent or Newton's method
        x_intermediate = update_points(x_intermediate, grad_L, step_size)
        
        # Check convergence
        if norm(grad_L) < tolerance:
            break
            
    # Check for bubble intersections
    if has_intersection_with_bubbles(points):
        return None # Path blocked
        
    return points, L
```

**Parameters:**
- `max_iterations`: Not given in the sources.
- `tolerance`: Not given in the sources.
- `step_size`: Not given in the sources.
- RIF Resolution: In the example, $1024 \times 1024 \times 2$ for the RIF [S1].
- Bubble Voxel Grid: $64 \times 64 \times 1$ [S1].
- Note: RIF resolution must be a multiple of the bubble voxel grid resolution to guarantee no missed intersections [S1].

**Edge Cases and Pitfalls:**
- Bubbles are considered opaque obstacles to limit complexity [S1].
- Multiple paths may exist between two points; all must be found for unbiasedness [S1].
- Precision issues may arise when enforcing $g_i(\mathbf{x}_i) = 0$ exactly [G].

## Subtopics

### [Fermat-Path-Tracing](20558-fermat-path-tracing.md)
A path-finding method that simulates any number of diffractions or reflections by minimizing the optical path length according to Fermat’s principle. It is distinguished from Min-Path-Tracing (MPT) by its specific objective function: while MPT minimizes a generic cost function, FPT explicitly minimizes the optical path length. This makes it suitable for unbiased estimation of transmitted lighting in complex environments involving refractive interfaces and heterogeneous media, building on the parent framework of [Unbiased Estimation of Transmitted Lighting](14564-unbiased-estimation-of-transmitted-lighting.md).

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Implemented in Mitsuba3 using the standard path tracing integrator with 1M samples per pixel [S1]. Rendered at $1024 \times 1024$ resolution [S1].
- **Strengths:** Unified estimation of surface and volume-induced caustics; handles heterogeneous media with analytical solutions for specific RIF forms [S1].
- **Limitations:** Bubbles are treated as opaque, limiting the ability to model transmission through bubbles; complexity of path estimation increases with the number of interfaces [S1].
- **When to use:** When unbiased estimation of transmitted caustics through complex refractive interfaces and heterogeneous media is required.
- **When not to use:** When performance is critical and approximate caustics are acceptable, or when bubbles need to be transparent.

## Background
The method is based on Fermat's principle, a fundamental principle in optics characterizing light ray trajectories [S1]. It was proposed to address the difficulty of finding direct paths between a receptor and a light source in the presence of refractive interfaces or media, where multiple paths may exist [S1]. The approach is similar in spirit to Manifold Walk but extends it to heterogeneous materials [S1]. It was implemented in Mitsuba3 and illustrated with a hand-blown cylinder glass panel [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
