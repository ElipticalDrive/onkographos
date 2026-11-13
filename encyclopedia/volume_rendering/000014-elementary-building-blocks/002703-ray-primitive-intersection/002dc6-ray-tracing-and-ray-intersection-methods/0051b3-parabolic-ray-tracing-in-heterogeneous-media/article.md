# Parabolic Ray Tracing in Heterogeneous Media

Also called: Non-linear ray tracing, Piecewise polynomial ray tracing

## Overview
Parabolic Ray Tracing is a method for computing light ray trajectories in media with spatially varying Refractive Index Fields (RIF) by approximating the RIF such that ray paths are second-order polynomials within tetrahedral cells [S1]. It solves the problem of exact intersection computation with cell boundaries, which is difficult when ray trajectories involve transcendental functions, by reducing the intersection problem to solving quadratic equations [S1]. It differs from linear ray tracing methods by accounting for continuous curvature within cells, and from general non-linear solvers by providing closed-form analytical solutions for the trajectory [S1].

## How it works
**Inputs:**
*   A tetrahedral mesh defining the spatial domain.
*   A Refractive Index Field (RIF) $\eta(\mathbf{x})$ defined per cell.
*   A ray origin $\mathbf{x}_0$ and initial direction $\mathbf{p}_0$ satisfying the Eikonal equation.

**Outputs:**
*   The trajectory of the ray as it traverses the medium.
*   Intersection points with cell boundaries.

**Data Structures:**
*   **Cell Coefficients:** For each tetrahedral cell, the squared RIF is defined as a linear function:
    $$ \eta^2(\mathbf{x}) = A_0 + A_1 \mathbf{x}_x + A_2 \mathbf{x}_y + A_3 \mathbf{x}_z = A_0 + \boldsymbol{\gamma} \cdot \mathbf{x} $$
    where $\boldsymbol{\gamma} = [A_1, A_2, A_3]^T$ is the constant gradient of the squared RIF, and $A_0$ is a scalar offset [S1].
*   **Ray State:** Position $\mathbf{x}(\tau)$ and direction $\mathbf{p}(\tau)$ at parameter $\tau$ [S1].

**Algorithm Steps:**

1.  **Initialize Ray State:**
    Set initial position $\mathbf{x}_0$ and initial direction $\mathbf{p}_0$. Ensure $\|\mathbf{p}_0\| = \eta(\mathbf{x}_0)$ [S1].

2.  **Determine Current Cell:**
    Identify the tetrahedral cell containing the current ray position $\mathbf{x}(\tau)$ [S1].

3.  **Compute Cell Coefficients:**
    Calculate $\boldsymbol{\gamma}$ and $A_0$ for the current cell based on the RIF definition [S1].

4.  **Compute Ray Trajectory within Cell:**
    The ray trajectory $\mathbf{x}(\tau)$ and direction $\mathbf{p}(\tau)$ are second-order polynomials in the parameter $\tau$:
    $$ \mathbf{x}(\tau) = \frac{1}{4}\boldsymbol{\gamma} \tau^2 + \mathbf{p}_0\tau + \mathbf{x}_0 $$
    $$ \mathbf{p}(\tau) = \frac{1}{2}\boldsymbol{\gamma} \tau + \mathbf{p}_0 $$
    where $\tau$ is a parameter related to arc length $s$ by $ds = \eta d\tau$ [S1].

5.  **Intersect with Cell Boundary:**
    Solve for the intersection of the parabolic ray subsection with the cell boundary. This is done by solving a quadratic equation exactly [S1].
    *   Let the cell boundary be defined by a plane equation $\mathbf{n} \cdot \mathbf{x} + d = 0$.
    *   Substitute $\mathbf{x}(\tau)$ into the plane equation:
        $$ \mathbf{n} \cdot \left( \frac{1}{4}\boldsymbol{\gamma} \tau^2 + \mathbf{p}_0\tau + \mathbf{x}_0 \right) + d = 0 $$
    *   Rearrange into standard quadratic form $a\tau^2 + b\tau + c = 0$:
        $$ a = \frac{1}{4} (\mathbf{n} \cdot \boldsymbol{\gamma}) $$
        $$ b = \mathbf{n} \cdot \mathbf{p}_0 $$
        $$ c = \mathbf{n} \cdot \mathbf{x}_0 + d $$
    *   Solve for $\tau$ using the quadratic formula. Select the smallest positive $\tau$ that corresponds to the exit point from the current cell [S1].

6.  **Update Ray State:**
    *   Update position: $\mathbf{x}_{new} = \mathbf{x}(\tau_{exit})$.
    *   Update direction: $\mathbf{p}_{new} = \mathbf{p}(\tau_{exit})$.
    *   Reset $\tau$ to 0 for the next cell, adjusting $\mathbf{x}_0$ and $\mathbf{p}_0$ accordingly [S1].

7.  **Repeat:**
    Repeat steps 2-6 until the ray exits the medium or reaches a termination condition [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def solve_quadratic(a, b, c):
    """Solve a*x^2 + b*x + c = 0. Returns smallest positive root or None."""
    if a == 0:
        if b == 0:
            return None
        root = -c / b
        return root if root > 0 else None
    discriminant = b * b - 4 * a * c
    if discriminant < 0:
        return None
    sqrt_disc = math.sqrt(discriminant)
    root1 = (-b - sqrt_disc) / (2 * a)
    root2 = (-b + sqrt_disc) / (2 * a)
    roots = [r for r in [root1, root2] if r > 0]
    return min(roots) if roots else None

def parabolic_ray_trace(ray_origin, ray_direction, tet_mesh, rif_function):
    """
    Trace a ray through a tetrahedral mesh with a piecewise linear squared RIF.
    
    Args:
        ray_origin: Initial position x0.
        ray_direction: Initial direction p0.
        tet_mesh: List of tetrahedra, each with vertices and plane equations for faces.
        rif_function: Function returning (gamma, A0) for a given position.
        
    Returns:
        List of intersection points.
    """
    x = ray_origin
    p = ray_direction
    intersections = []
    
    # Ensure initial direction satisfies Eikonal equation
    # p_norm = norm(p)
    # eta_x0 = rif_function(x)[1] # A0 is not eta, need to compute eta from A0 + gamma.x
    # eta_x0 = math.sqrt(A0 + dot(gamma, x))
    # p = p * (eta_x0 / p_norm) # Normalize to match eta
    
    current_tet = find_tetrahedron_containing_point(tet_mesh, x)
    
    while current_tet is not None:
        gamma, A0 = rif_function(x) # Get coefficients for current cell
        
        # Find exit point from current tetrahedron
        min_tau = float('inf')
        exit_face = None
        
        for face in current_tet.faces:
            # Plane equation: n . x + d = 0
            n = face.normal
            d = -dot(n, face.vertices[0])
            
            # Coefficients for quadratic equation
            a = 0.25 * dot(n, gamma)
            b = dot(n, p)
            c = dot(n, x) + d
            
            tau = solve_quadratic(a, b, c)
            
            if tau is not None and tau < min_tau:
                min_tau = tau
                exit_face = face
        
        if min_tau == float('inf'):
            # Ray stays within tetrahedron (should not happen in convex mesh)
            break
            
        # Compute exit position and direction
        exit_x = 0.25 * gamma * (min_tau ** 2) + p * min_tau + x
        exit_p = 0.5 * gamma * min_tau + p
        
        intersections.append(exit_x)
        
        # Update state for next cell
        x = exit_x
        p = exit_p
        
        # Find next tetrahedron
        current_tet = find_adjacent_tetrahedron(tet_mesh, current_tet, exit_face)
        
    return intersections
```

**Edge Cases and Pitfalls:**
*   **Precision:** Solving the quadratic equation exactly is crucial for boundary intersection accuracy [S1].
*   **Eikonal Equation:** The initial direction must satisfy $\|\mathbf{p}_0\| = \eta(\mathbf{x}_0)$ to ensure physical correctness [S1].
*   **Degenerate Input:** If the ray is tangent to a cell boundary, the quadratic equation may have a double root or no positive root, requiring careful handling [S1].

## Operating characteristics
*   **Time Complexity:** Linear in the number of cells traversed, with constant time per cell for intersection computation (solving a quadratic equation) [S1].
*   **Memory Complexity:** Linear in the size of the tetrahedral mesh and RIF data [S1].
*   **Strengths:** Exact intersection computation with cell boundaries; analytical solutions for ray trajectories within cells [S1].
*   **Limitations:** Limited to media where the squared RIF can be approximated as a linear function within tetrahedral cells; may require fine mesh resolution for accurate RIF representation [S1].
*   **When to Use:** When accurate ray tracing in heterogeneous media with smooth refractive index variations is required, and the medium can be discretized into tetrahedral cells with linear squared RIF profiles [S1].
*   **When Not to Use:** When the RIF varies rapidly or non-linearly within cells, or when the medium is not naturally represented as a tetrahedral mesh [S1].

## Background
Parabolic Ray Tracing was introduced in the context of seismic wave propagation, where Virieux and Farra demonstrated that a well-chosen expression for the slowness profile within tetrahedral cells yields simple second-order polynomial solutions for ray trajectories [S1]. This approach was later adapted for computer graphics to model light propagation in heterogeneous media, such as old glass, by assembling multiple tetrahedral cells and tracing piecewise polynomial rays [S1]. It generalizes the concept of ray differentials used in computer graphics for building two-point tracing methods [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
