# Barycentric Tetrahedral Isosurface Ray Intersection

Also called: Unstructured mesh ray intersection

## Overview
This method intersects a ray with an isosurface implicitly defined within a tetrahedral cell of an unstructured mesh using barycentric interpolation [S1]. It solves for the intersection parameter $t$ by reducing the density constraint to a linear equation, distinguishing it from methods that may require quadratic solutions or explicit surface reconstruction [S1].

## How it works
**Inputs and Outputs**
*   **Input**: Ray origin $\mathbf{a}$, ray direction $\vec{b}$ (assumed normalized), tetrahedron vertices $\mathbf{p}_i = (x_i, y_i, z_i)$ for $i=0..3$, vertex densities $\rho_i$, and isovalue $\rho_{\text{iso}}$.
*   **Output**: Intersection parameter $t$, intersection point $\mathbf{p}(t)$, and normal vector $\vec{\mathbf{N}}$. If no valid intersection exists, the method returns failure.

**Data Structures**
The tetrahedron is defined by four vertices. Each vertex $\mathbf{p}_i$ has associated scalar density $\rho_i$. Precomputed plane equations $f_i(\mathbf{p})$ are required for efficient barycentric coordinate calculation.

**Plane Equation Definition**
For each face $i$ opposite vertex $\mathbf{p}_i$, define a plane equation $f_i(\mathbf{p})$ such that $f_i(\mathbf{p}_i) = 1$ and $f_i(\mathbf{p}_j) = 0$ for $j \neq i$. The equation takes the form:
$$ f_i(\mathbf{p}) = \vec{\mathbf{n}}_i \cdot (\mathbf{p} - \mathbf{q}_i) $$
where $\vec{\mathbf{n}}_i$ is the normal vector and $\mathbf{q}_i$ is a point on the plane. The barycentric coordinate $\alpha_i$ for any point $\mathbf{p}$ is simply $f_i(\mathbf{p})$ [S1].

**Steps**
1.  **Precompute Plane Parameters**: For each vertex $i$, determine $\vec{\mathbf{n}}_i$ and $\mathbf{q}_i$ such that $f_i(\mathbf{p}) = \vec{\mathbf{n}}_i \cdot (\mathbf{p} - \mathbf{q}_i)$ satisfies the barycentric conditions. Note that $\vec{\mathbf{n}}_i$ is constant for the tetrahedron [S1].
2.  **Formulate Linear Equation**: Substitute the ray equation $\mathbf{p}(t) = \mathbf{a} + t\vec{b}$ into the density interpolation formula:
    $$ \rho(t) = \sum_{i=0}^3 f_i(\mathbf{a} + t\vec{b})\rho_i = \rho_{\text{iso}} $$
    Since $f_i(\mathbf{a} + t\vec{b}) = \vec{\mathbf{n}}_i \cdot (\mathbf{a} + t\vec{b} - \mathbf{q}_i) = \vec{\mathbf{n}}_i \cdot (\mathbf{a} - \mathbf{q}_i) + t (\vec{\mathbf{n}}_i \cdot \vec{b})$, the equation becomes linear in $t$:
    $$ t \left( \sum_{i=0}^3 \rho_i (\vec{\mathbf{n}}_i \cdot \vec{b}) \right) + \sum_{i=0}^3 \rho_i (\vec{\mathbf{n}}_i \cdot (\mathbf{a} - \mathbf{q}_i)) = \rho_{\text{iso}} $$
    Let $A = \sum_{i=0}^3 \rho_i (\vec{\mathbf{n}}_i \cdot \vec{b})$ and $B = \sum_{i=0}^3 \rho_i (\vec{\mathbf{n}}_i \cdot (\mathbf{a} - \mathbf{q}_i)) - \rho_{\text{iso}}$. The equation is $A t + B = 0$ [S1].
3.  **Solve for $t$**:
    *   If $|A| < \epsilon$ (where $\epsilon$ is a small epsilon, e.g., $10^{-30}$ for stability with large coefficients), the ray is parallel to the isosurface gradient or the solution is degenerate. Handle as no intersection or special case [S1].
    *   Otherwise, $t = -B / A$ [S1].
4.  **Validate Intersection**: Calculate the intersection point $\mathbf{p}(t) = \mathbf{a} + t\vec{b}$. Compute barycentric coordinates $\alpha_i = f_i(\mathbf{p}(t))$ for all $i=0..3$. If $\alpha_i > 0$ for all $i$, the intersection is inside the tetrahedron and is accepted. Otherwise, reject [S1].
5.  **Compute Normal**: If accepted, compute the normal vector $\vec{\mathbf{N}}$ as:
    $$ \vec{\mathbf{N}} = \sum_{i=0}^3 \rho_i \vec{\mathbf{n}}_i $$
    This normal is constant within the tetrahedron for linear barycentric interpolation [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barycentric_tetra_isosurface_intersect(ray_origin, ray_dir, tetra_vertices, vertex_densities, rho_iso):
    # Precomputed plane normals and points for the tetrahedron
    # n[i] is the normal vector for face i
    # q[i] is a point on face i
    # These must be precomputed such that f_i(p) = dot(n[i], p - q[i])
    # and f_i(p_j) = 1 if i==j else 0
    
    A = 0.0
    B = 0.0
    
    for i in range(4):
        # Calculate dot products
        n_dot_b = dot(tetra_plane_normals[i], ray_dir)
        n_dot_a_minus_q = dot(tetra_plane_normals[i], ray_origin - tetra_plane_points[i])
        
        A += vertex_densities[i] * n_dot_b
        B += vertex_densities[i] * n_dot_a_minus_q
        
    B -= rho_iso
    
    # Check for degenerate case where A is near zero
    if abs(A) < 1e-30:
        return None  # No unique solution or parallel
        
    t = -B / A
    
    # Calculate intersection point
    p = ray_origin + t * ray_dir
    
    # Check if point is inside tetrahedron using barycentric coordinates
    for i in range(4):
        alpha_i = dot(tetra_plane_normals[i], p - tetra_plane_points[i])
        if alpha_i <= 0:
            return None  # Outside tetrahedron
            
    # Compute normal
    normal = vec3(0, 0, 0)
    for i in range(4):
        normal += vertex_densities[i] * tetra_plane_normals[i]
        
    return {
        't': t,
        'point': p,
        'normal': normalize(normal)
    }
```

**Parameters**
*   $\epsilon$: Epsilon for stability check. Source suggests $10^{-30}$ for maximum stability with large coefficients [S1].

**Edge Cases and Pitfalls**
*   **Degenerate $A$**: If $A \approx 0$, the ray direction is orthogonal to the effective gradient of the density field within the cell, or the density variation is negligible. The source code modifies special cases for linear or quadratic solutions, assuming $A$ is nonzero in the standard derivation [S1].
*   **Precision**: Use high-precision arithmetic for plane equation evaluations if coefficients are large to avoid numerical instability [S1].

## Subtopics

### [Fast Convex Polygon Wrapper (FCPW) Accelerator](1664-fast-convex-polygon-wrapper-fcpw-accelerator.md)
Uses convex hulls to accelerate ray traversal, differing in its use of explicit polygonal approximations rather than implicit barycentric interpolation.

### [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md)
Uses Plücker coordinates for intersection tests, offering a different algebraic approach to ray-tetrahedron intersection.

### [Point-in-Tetrahedron Test via Barycentric Coordinates](2591-point-in-tetrahedron-test-via-barycentric-coordinates.md)
Provides the fundamental geometric test used in step 4 of this method.

### [Tetrahedron Face Extraction](3742-tetrahedron-face-extraction.md)
Details the geometry of tetrahedron faces, relevant for precomputing plane equations.

### [Skeleton Parameter Minimization for Ray Intersection](3975-skeleton-parameter-minimization-for-ray-intersection.md)
Optimizes ray intersection by reducing the search space, a complementary optimization technique.

### [Deformed element ray intersection](5148-deformed-element-ray-intersection.md)
Handles non-linear or deformed elements, whereas this method assumes linear barycentric interpolation.

### [Mesh Shader Tetrahedron Intersection Optimization](5174-mesh-shader-tetrahedron-intersection-optimization.md)
Focuses on GPU-specific optimizations for this type of intersection.

### [Raytracing Cell-Organized Irregular Volumes](5257-raytracing-cell-organized-irregular-volumes.md)
Discusses acceleration structures for unstructured meshes, which this method operates on.

### [Point Calculus Faceted Solid Raycasting](18547-point-calculus-faceted-solid-raycasting.md)
An alternative approach to raycasting on unstructured grids using point calculus.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per tetrahedron, involving a constant number of dot products and arithmetic operations.
*   **Memory Complexity**: $O(1)$ auxiliary memory per tetrahedron, assuming plane parameters are precomputed and stored with the mesh.
*   **Strengths**: Simple linear solution; robust with proper epsilon handling; naturally provides normals.
*   **Limitations**: Assumes linear density variation within the tetrahedron; requires precomputed plane equations; sensitive to numerical precision if coefficients are large [S1].
*   **When to Use**: Ideal for unstructured tetrahedral meshes with linearly interpolated scalar fields where isosurface extraction is required.
*   **When Not to Use**: Not suitable for higher-order interpolation schemes without modification; less efficient than hierarchical acceleration structures for large volumes.

## Background
This method is based on the work of Schwarze, as cited in the source [S1]. The approach was implemented with modifications for stability, specifically handling cases where the linear coefficient $A$ might be zero and using a very small epsilon ($10^{-30}$) for numerical stability [S1]. It is part of the broader field of interactive ray tracing for volume visualization on unstructured meshes.

## Sources
[S1] Interactive Ray Tracing for Volume Visualization, https://www.ppsloan.org/publications/tvcg99.pdf, fetched 2026-10-29
