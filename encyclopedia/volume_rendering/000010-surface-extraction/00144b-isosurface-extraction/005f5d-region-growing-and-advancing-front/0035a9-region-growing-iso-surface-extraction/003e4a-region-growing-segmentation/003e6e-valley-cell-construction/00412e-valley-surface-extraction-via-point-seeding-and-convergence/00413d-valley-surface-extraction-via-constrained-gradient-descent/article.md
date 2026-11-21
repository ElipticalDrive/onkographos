# Valley Surface Extraction via Constrained Gradient Descent

Also called: Gradient descent in non-manifold regions, Slope-line extraction

## Overview
A mesh-free surface extraction method that extracts valley surfaces (maximal convexity ridges) in complex scalar fields by handling non-manifold regions where standard surface growing fails [S1, unofficial]. It identifies branching locations (partial umbilics) and connects them using constrained gradient-based integration, specifically addressing areas of ambiguous convexity where the gradient has a distinct and consistent direction [S1, unofficial]. Unlike flow-based surface integration, which is numerically unstable due to substantially varying gradient directions close to separation structures, this approach uses constrained gradient descent only in these ambiguous regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field $f$, an existing partial valley surface (point cloud with connectivity), and a convergence tolerance $\epsilon$ [S1, unofficial].
*   **Output:** An extended point cloud representing the connected valley surface graph, including connector structures in non-manifold regions [S1, unofficial].

**Data Structures:**
*   **Surface Point:** Contains position $\mathbf{p}$, normal $\mathbf{n}$, and curvature information (eigenvalues $\lambda_1, \lambda_2, \lambda_3$ and eigenvectors $\mathbf{e}_1, \mathbf{e}_2, \mathbf{e}_3$ of the Hessian) [S1, unofficial].
*   **Connectivity:** Neighborhood relations between points to guide surface growth and triangulation [S1, unofficial].

**Definitions:**
*   $\lambda_1, \lambda_2, \lambda_3$: Eigenvalues of the Hessian matrix, ordered such that $\lambda_1 \le \lambda_2 \le \lambda_3$. $\lambda_3$ corresponds to the direction of maximal convexity (valley direction), $\lambda_2$ to medium curvature, and $\lambda_1$ to minimal curvature [S1, unofficial].
*   $\mathbf{e}_1, \mathbf{e}_2, \mathbf{e}_3$: Corresponding eigenvectors.
*   $\nabla f$: Gradient of the scalar field.
*   $\nabla f_p$: Gradient projected onto the plane of convergence [S1, unofficial].
*   $\delta$: Threshold for identifying partial umbilics [S1, unofficial].

**Steps:**

1.  **Identify Partial Umbilics:** Scan the boundary of the existing surface to find locations where $\frac{\lambda_2}{\lambda_3} > 1 - \delta$ and a change in main curvature direction was detected during previous convergence steps [S1, unofficial]. These are branching locations [S1, unofficial].

2.  **Handle Degenerate Line Boundaries (Case 1):** If new child points fail to converge due to a degenerate line boundary, the inverse gradient direction points towards the branching region, describing a flow separation line [S1, unofficial].
    *   Constrain the gradient descent to a plane orthogonal to the direction of minimal curvature ($\mathbf{e}_1$) [S1, unofficial].
    *   On this projection plane, the vector field degenerates to a source point where the crease line intersects [S1, unofficial].
    *   Step along the negative gradient direction as long as $\lambda_2 > 0$ until meeting the maximal convexity ridge at a crease line point [S1, unofficial].
    *   Merge the created start and end positions of these connector lines into the existing valley structure and update point neighborhood relations [S1, unofficial].

3.  **Handle Junction-like Regions (Case 2):** If the surface continues growing tangentially to the minimal and medium curvature directions and crosses the junction-like region, it indicates a nearby valley surface [S1, unofficial].
    *   Seed a set of points in the neighborhood to ensure all nearby structures are recovered by the extraction algorithm [S1, unofficial].
    *   These seeded points require an additional convergence step to meet the valley surface [S1, unofficial].

4.  **Extract Connector Structures near Degenerate Lines:** If $\lambda_2 \gg 0$ close to a degenerate line, extract the connector structure by replacing $\lambda_3$ in the standard valley line equation (referenced as Eq. 4.2 in source) with $\lambda_2$ [S1, unofficial]. This merges corresponding crease lines with degenerate lines [S1, unofficial].

5.  **Convergence:** Use constrained Newton iteration or quadratic approximation to find the field minimum on the plane spanned by $\mathbf{e}_2$ and $\mathbf{e}_3$ [S1, unofficial]. The projected gradient $\nabla f_p$ is used for this optimization [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_valley_surface(surface_points, scalar_field, epsilon, delta):
    """
    Extracts valley surfaces using constrained gradient descent in non-manifold regions.
    
    Args:
        surface_points: List of Point objects with pos, normal, eigenvalues (l1, l2, l3), 
                        eigenvectors (e1, e2, e3).
        scalar_field: Function f(p) returning scalar value at position p.
        epsilon: Convergence tolerance.
        delta: Threshold for partial umbilic detection.
        
    Returns:
        Updated list of surface_points.
    """
    new_points = []
    
    for point in surface_points:
        # Check for partial umbilics on boundary
        if is_boundary(point) and is_partial_umbilic(point, delta):
            # Identify branching location
            branching_location = point.pos
            
            # Case 1: Handle degenerate line boundary
            if fails_to_converge(point, scalar_field):
                # Constrain descent to plane orthogonal to e1 (minimal curvature)
                plane_normal = point.e1
                plane_origin = point.pos
                
                # Project gradient to plane
                grad = compute_gradient(point.pos, scalar_field)
                grad_proj = project_vector(grad, plane_normal)
                
                # Step along negative gradient as long as lambda2 > 0
                current_pos = point.pos
                while point.l2 > 0:
                    # Find minimum on plane spanned by e2 and e3
                    # Using quadratic approximation or Newton iteration
                    next_pos = constrained_minimize(current_pos, scalar_field, plane_normal, plane_origin, epsilon)
                    
                    # Update point properties at new position
                    new_point = compute_point_properties(next_pos, scalar_field)
                    new_points.append(new_point)
                    
                    # Check if crease line point is met
                    if is_crease_line_point(new_point):
                        break
                    
                    current_pos = next_pos
                    # Recompute eigenvalues for loop condition
                    point = new_point 
                
                # Merge start and end positions into structure
                merge_connector_lines(point.pos, current_pos, surface_points)
                
            # Case 2: Handle junction-like region
            elif crosses_junction(point, scalar_field):
                # Seed points in neighborhood
                neighborhood_points = seed_points_in_neighborhood(point.pos, scalar_field)
                new_points.extend(neighborhood_points)
                
                # Additional convergence step for seeded points
                for np in neighborhood_points:
                    converge_to_valley(np, scalar_field, epsilon)
                    
            # Connector structure extraction near degenerate lines
            if is_close_to_degenerate_line(point) and point.l2 >> 0:
                # Replace lambda3 with lambda2 in valley line equation
                connector = extract_connector_with_lambda2(point.pos, scalar_field, point.l2)
                new_points.append(connector)
                
    return surface_points + new_points

def is_partial_umbilic(point, delta):
    """
    Checks if point is a partial umbilic: lambda2/lambda3 > 1 - delta
    and change in main curvature direction detected.
    """
    if point.l3 == 0:
        return False
    ratio = point.l2 / point.l3
    return ratio > (1 - delta) and has_curvature_direction_change(point)

def constrained_minimize(start_pos, scalar_field, plane_normal, plane_origin, epsilon):
    """
    Finds the minimum of the scalar field on the plane defined by normal and origin.
    Uses quadratic approximation if possible, otherwise Newton iteration.
    """
    # Project start_pos to plane
    projected_pos = project_point_to_plane(start_pos, plane_normal, plane_origin)
    
    # Iterative minimization on the 2D plane
    current_pos = projected_pos
    while True:
        grad = compute_gradient(current_pos, scalar_field)
        grad_proj = project_vector(grad, plane_normal)
        
        if norm(grad_proj) < epsilon:
            break
            
        # Newton step or gradient descent step on the plane
        # Hessian H is computed at current_pos
        H = compute_hessian(current_pos, scalar_field)
        # Project Hessian to plane basis (e2, e3)
        H_proj = project_hessian(H, plane_normal)
        
        # Solve for step in plane coordinates
        step = solve_linear_system(H_proj, -grad_proj)
        current_pos = current_pos + step
        
    return current_pos

def project_vector(v, normal):
    """Projects vector v onto the plane orthogonal to normal."""
    return v - dot(v, normal) * normal

def project_point_to_plane(p, normal, origin):
    """Projects point p onto the plane defined by origin and normal."""
    t = dot(p - origin, normal)
    return p - t * normal

def compute_gradient(pos, scalar_field):
    """Computes gradient using finite differences."""
    # Implementation details omitted for brevity, assumed standard central differences
    pass

def compute_hessian(pos, scalar_field):
    """Computes Hessian matrix using finite differences."""
    # Implementation details omitted for brevity
    pass

def solve_linear_system(A, b):
    """Solves Ax = b for x."""
    # Use standard linear algebra solver
    pass

def is_crease_line_point(point):
    """Checks if point lies on a crease line."""
    # Criterion based on Kenwright et al. [KHL99]
    # Gradient field parallel to minimal eigenvector of Hessian
    return is_parallel_to_eigenvector(point, point.e1)

def is_parallel_to_eigenvector(point, eigenvector):
    """Checks if gradient is parallel to eigenvector."""
    grad = compute_gradient(point.pos, scalar_field)
    return is_parallel(grad, eigenvector)

def is_parallel(v1, v2):
    """Checks if two vectors are parallel."""
    return abs(dot(normalize(v1), normalize(v2))) > 0.99
```

**Parameters:**
*   $\epsilon$: Convergence tolerance. Typical value: $10^{-8}$ [S1, unofficial].
*   $\delta$: Threshold for partial umbilic detection. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Flat Medium Curvature:** The extraction technique is numerically not reliable in regions where the field is flat in the direction of medium curvature ($\lambda_2 \approx 0$), as this causes deviations in the approximated medium curvature direction [S1, unofficial].
*   **Noisy Data:** Surface quality is reduced in noisy data sets due to less accurate curvature and ridge normal computations [S1, unofficial].
*   **Resolution Artifacts:** Decreasing resolution causes curved areas to show edge artifacts due to lack of curvature adaptivity and increased edge lengths [S1, unofficial].

## Subtopics

### [Triangulation of Valley Surfaces](10463-triangulation-of-valley-surfaces.md)
A procedure to triangulate the point cloud representing a connected surface graph created by the extraction algorithm, using local Delaunay triangles and specific handling for non-manifold regions bounded by crease and degenerate lines. It uses the neighborhood relations from the extraction phase to create local Delaunay triangles, ensuring that crease lines form edges in the triangulation via edge swapping.

### Interactive Surface Growing and Refinement
A technique for selective surface construction and refinement, allowing interactive update of the valley surface [S1, unofficial]. It uses a view-aligned cylindrical selection tool to mark regions for refinement or growing, with visual highlighting performed by OpenGL stencil-buffering [S1, unofficial]. Decreasing resolution causes curved areas to show edge artifacts due to lack of curvature adaptivity and increased edge lengths [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Not given in the sources.

**Measured Performance:**
*   Extraction times and point numbers for different levels of detail are provided in Table 4.1 of [S1, unofficial].
    *   **L-shape dataset:**
        *   High res: 112s, 4780 pts
        *   Medium res: 66.8s, 3040 pts
        *   Low res: 56.4s, 1760 pts
    *   **137 Hz dataset:**
        *   High res: 48s, 4270 pts
        *   Medium res: 15.9s, 1310 pts
        *   Low res: 12.2s, 880 pts
*   These results use $\epsilon = 10^{-8}$ and constrained Newton iteration [S1, unofficial].

**Scaling:**
*   Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Numerically stable in areas of ambiguous convexity where the gradient has a distinct and consistent direction [S1, unofficial]. Allows selective surface construction and refinement [S1, unofficial]. Extracted meshes are highly parallel to the true ridge surfaces [S1, unofficial].
*   **Limitations:** Numerically unreliable in regions where the field is flat in the direction of medium curvature [S1, unofficial]. Surface quality is reduced in noisy data sets [S1, unofficial]. Edge artifacts appear at low resolution [S1, unofficial].

**When to Use:**
*   Use for extracting valley surfaces in complex scalar fields with non-manifold regions, branching locations, and junction-like structures [S1, unofficial].

**When Not to Use:**
*   Avoid in regions where the field is flat in the direction of medium curvature [S1, unofficial]. Avoid for noisy data sets if high surface quality is required [S1, unofficial].

## Background
**Origin:**
*   The method is described in the thesis "Multi-Field Visualization" by [S1, unofficial].

**History and Development:**
*   The method builds on previous work in mesh-free valley surface extraction [S1, unofficial].
*   Numerical procedures for finding stable degenerate lines in grid-based fields are presented in [ZP04] (cited in [S1, unofficial]).
*   The criterion for separation and attachment lines is based on Kenwright et al. [KHL99] (cited in [S1, unofficial]).

**Where it is Used:**
*   Used in multi-field visualization for extracting complex surface structures in scalar fields [S1, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
