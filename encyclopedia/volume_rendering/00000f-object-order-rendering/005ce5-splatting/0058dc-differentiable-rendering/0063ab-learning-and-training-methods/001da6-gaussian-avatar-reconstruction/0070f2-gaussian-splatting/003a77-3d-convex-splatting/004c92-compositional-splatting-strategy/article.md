# Compositional Splatting Strategy

Also called: Compositional Splatting, Gaussian Line Splatting, Gaussian lines, Line-segment-centered primitive splatting, MP-GS Compositional Splatting

## Overview
Compositional Splatting is a rendering strategy for non-elliptical primitives, specifically Gaussian lines and Gaussian triangles, which decomposes the primitive into its constituent vertices, splats them individually using point-based techniques, and re-sketches the final shape on the image plane [S1, unofficial]. It solves the problem of rendering primitives that lack a simple elliptical boundary, enabling mixed-primitive Gaussian Splatting frameworks to accurately reconstruct object surfaces [S1, unofficial]. Unlike standard Gaussian Splatting which renders ellipses directly, this method constructs the primitive's silhouette from the common tangents of vertex-based boundary ellipses [S1, unofficial].

## How it works
The method renders Gaussian lines and Gaussian triangles by treating them as compositions of their vertices [S1, unofficial]. The process involves primitive formulation, compositional splatting of vertices, boundary ellipse formulation, common tangent calculation, and modified alpha blending [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Primitive type: Gaussian line (2 vertices) or Gaussian triangle (3 vertices) [S1, unofficial].
    *   Viewpoint parameters: Projection matrix $P$, world-to-view transformation $W$, and Jacobian $J$ [S1, unofficial].
    *   Primitive parameters:
        *   Gaussian Line: Two 3D vertex coordinates $\mu_1, \mu_2$ [S1, unofficial].
        *   Gaussian Triangle: First vertex 3D coordinate $\mu_1 \in \mathbb{R}^3$, second and third vertices 2D relative coordinates $\mu_2, \mu_3 \in \mathbb{R}^2$, rotation matrix $R$, and shared covariance matrix $\Sigma$ [S1, unofficial].
    *   Rendering parameters: Opacity $\alpha$, color $c$ [S1, unofficial].
*   **Outputs**:
    *   Image plane pixels with color and opacity values derived from the re-sketched primitive [S1, unofficial].

### Data Structures
**Gaussian Triangle**:
*   $\mu_1$: $\mathbb{R}^3$ (3D coordinate of the first vertex).
*   $\mu_2$: $\mathbb{R}^2$ (2D relative coordinate of the second vertex in the triangle's local plane).
*   $\mu_3$: $\mathbb{R}^2$ (2D relative coordinate of the third vertex in the triangle's local plane).
*   $R$: $3 \times 3$ rotation matrix defining the triangle's orientation.
*   $\Sigma$: $3 \times 3$ covariance matrix (shared by all three vertices).

**Gaussian Line**:
*   $\mu_1, \mu_2$: $\mathbb{R}^3$ (3D coordinates of the two endpoints).
*   $\Sigma$: $3 \times 3$ covariance matrix (shared by both vertices).

### Numbered Steps

**Step 1: Primitive Formulation (3D to Local 2D)** [S1, unofficial]. For a Gaussian triangle, reconstruct the 3D coordinates of the second and third vertices using the first vertex and the rotation matrix: [S1, unofficial]
$$
\mu_2^{3D} = \mu_1 + \mu_2[0] \times (R[:, 0]^T) + \mu_2[1] \times (R[:, 1]^T)
$$
$$
\mu_3^{3D} = \mu_1 + \mu_3[0] \times (R[:, 0]^T) + \mu_3[1] \times (R[:, 1]^T)
$$
For a Gaussian line, the vertices are already in 3D space [S1, unofficial].

**Step 2: Compositional Splatting (Projection)** [S1, unofficial]. Project each vertex and the shared covariance to the 2D image plane [S1, unofficial]. Let $P$ be the projection matrix and $W$ be the world-to-view transformation [S1, unofficial].
$$
\mu_1^{2D} = (PW\mu_1)[:2]
$$
$$
\mu_2^{2D} = (PW\mu_2^{3D})[:2] \quad (\text{or } (PW\mu_2)[:2] \text{ for lines})
$$
$$
\mu_3^{2D} = (PW\mu_3^{3D})[:2] \quad (\text{for triangles})
$$
$$
\Sigma^{2D} = (J W \Sigma W^T J^T)[:2, :2]
$$

**Step 3: Re-sketching on the Image Plane** [S1, unofficial]. This step constructs the boundary of the primitive on the image plane [S1, unofficial].

**Step 3.1: Boundary Ellipse Formulation** [S1, unofficial]. Define the boundary ellipse for each vertex $i$ as the contour where the Gaussian value is $1/255$ [S1, unofficial]. The contour is the set of points $(x,y)$ satisfying $g_i^{ct} = 0$: [S1, unofficial]
$$
g_i^{ct} = -2 \ln(255) + \frac{c_\Sigma(x_{\mu_i} - x)^2 + a_\Sigma(y_{\mu_i} - y)^2 + b_\Sigma(x_{\mu_i} - x)(y_{\mu_i} - y)}{a_\Sigma c_\Sigma - b_\Sigma^2} = 0
$$
Here, $\Sigma^{2D}$ is decomposed into symmetric matrix elements $a_\Sigma, b_\Sigma, c_\Sigma$ [S1, unofficial]. The equation is expanded into the general quadratic form: [S1, unofficial]
$$
g_i^{ct} = a_i x^2 + b_i xy + c_i y^2 + d_i x + e_i y + f_i = 0
$$
The coefficients $a_i, b_i, c_i, d_i, e_i, f_i$ are derived from $\mu_i^{2D}$ and $\Sigma^{2D}$ [S1, unofficial].

**Step 3.2: Common Tangent Line Measurement** [S1, unofficial]. Calculate the common tangent lines between pairs of boundary ellipses [S1, unofficial]. This is done by finding the intersection of their dual ellipses [S1, unofficial].
1.  Compute the dual ellipse function $g_i^{dual}$ for each boundary ellipse using projective geometry duality: [S1, unofficial]
    $$
    g_i^{dual} = \left[ -\frac{1}{4}(e_i)^2 + c_i f_i \right] x^2 + \left[ \frac{1}{2} d_i e_i - b_i f_i \right] xy + \left[ -\frac{1}{4}(d_i)^2 + a_i f_i \right] y^2 + \left[ -c_i d_i + \frac{1}{2} b_i e_i \right] x + \left[ \frac{1}{2} b_i e_i - a_i e_i \right] y - \frac{1}{4}(b_i)^2 + a_i c_i = 0
    $$
2.  Solve the system $g_1^{dual}=0, g_2^{dual}=0$ for the intersection point $(x_s, y_s)$ [S1, unofficial].
3.  The common tangent line is given by: [S1, unofficial]
    $$
    x_s x + y_s y + 1 = 0
    $$
    This calculation has $O(1)$ time complexity [S1, unofficial].

**Step 4: Rendering** [S1, unofficial]. Perform image rendering over the re-sketched primitives using a modified alpha-blending function [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compositional_splatting(primitive_type, vertices_3d, rotation_matrix, covariance_3d, P, W, J):
    """
    Renders a Gaussian line or triangle using compositional splatting.
    
    Args:
        primitive_type: 'line' or 'triangle'
        vertices_3d: List of coordinates. For line: [mu1_3d, mu2_3d].
                     For triangle: [mu1_3d, mu2_2d_rel, mu3_2d_rel]
        rotation_matrix: 3x3 R matrix (required for triangle)
        covariance_3d: 3x3 Sigma matrix
        P: Projection matrix
        W: World-to-view transformation matrix
        J: Jacobian of the projective transformation
        
    Returns:
        Rendered image contribution (color, opacity) for the primitive.
    """
    # Step 1: Formulate 3D coordinates for all vertices
    if primitive_type == 'triangle':
        mu1_3d = vertices_3d[0]
        mu2_rel = vertices_3d[1]
        mu3_rel = vertices_3d[2]
        
        mu2_3d = mu1_3d + mu2_rel[0] * rotation_matrix[:, 0] + mu2_rel[1] * rotation_matrix[:, 1]
        mu3_3d = mu1_3d + mu3_rel[0] * rotation_matrix[:, 0] + mu3_rel[1] * rotation_matrix[:, 1]
        
        vertex_3ds = [mu1_3d, mu2_3d, mu3_3d]
    else: # line
        mu1_3d = vertices_3d[0]
        mu2_3d = vertices_3d[1]
        vertex_3ds = [mu1_3d, mu2_3d]
        
    num_vertices = len(vertex_3ds)
    
    # Step 2: Compositional Splatting (Project to 2D)
    vertex_2ds = []
    for mu_3d in vertex_3ds:
        mu_2d = (P @ W @ mu_3d)[:2]
        vertex_2ds.append(mu_2d)
        
    # Project Covariance to 2D
    W_sigma_WT = W @ covariance_3d @ W.T
    J_W_sigma_WT_JT = J @ W_sigma_WT @ J.T
    sigma_2d = J_W_sigma_WT_JT[:2, :2]
    
    # Step 3: Re-sketching
    # Step 3.1: Boundary Ellipse Formulation
    boundary_ellipses = []
    for i in range(num_vertices):
        mu_i = vertex_2ds[i]
        a_sig = sigma_2d[0, 0]
        b_sig = sigma_2d[0, 1]
        c_sig = sigma_2d[1, 1]
        
        denom = a_sig * c_sig - b_sig**2
        log_term = -2 * math.log(255)
        
        # Expand g_i^ct = 0 to a_i*x^2 + b_i*xy + c_i*y^2 + d_i*x + e_i*y + f_i = 0
        a_i = c_sig / denom
        c_i = a_sig / denom
        b_i = b_sig / denom
        d_i = (-2 * c_sig * mu_i[0] - b_sig * mu_i[1]) / denom
        e_i = (-2 * a_sig * mu_i[1] - b_sig * mu_i[0]) / denom
        f_i = log_term + (c_sig * mu_i[0]**2 + a_sig * mu_i[1]**2 + b_sig * mu_i[0] * mu_i[1]) / denom
        
        boundary_ellipses.append({'a': a_i, 'b': b_i, 'c': c_i, 'd': d_i, 'e': e_i, 'f': f_i})
        
    # Step 3.2: Common Tangent Line Measurement
    tangent_lines = []
    if num_vertices == 2:
        pairs = [(0, 1)]
    else: # triangle
        pairs = [(0, 1), (1, 2), (2, 0)]
        
    for i, j in pairs:
        ell_i = boundary_ellipses[i]
        ell_j = boundary_ellipses[j]
        
        # Calculate dual ellipse coefficients for i
        A_i = -0.25 * ell_i['e']**2 + ell_i['c'] * ell_i['f']
        B_i = 0.5 * ell_i['d'] * ell_i['e'] - ell_i['b'] * ell_i['f']
        C_i = -0.25 * ell_i['d']**2 + ell_i['a'] * ell_i['f']
        D_i = -ell_i['c'] * ell_i['d'] + 0.5 * ell_i['b'] * ell_i['e']
        E_i = 0.5 * ell_i['b'] * ell_i['e'] - ell_i['a'] * ell_i['e']
        F_i = -0.25 * ell_i['b']**2 + ell_i['a'] * ell_i['c']
        
        # Calculate dual ellipse coefficients for j
        A_j = -0.25 * ell_j['e']**2 + ell_j['c'] * ell_j['f']
        B_j = 0.5 * ell_j['d'] * ell_j['e'] - ell_j['b'] * ell_j['f']
        C_j = -0.25 * ell_j['d']**2 + ell_j['a'] * ell_j['f']
        D_j = -ell_j['c'] * ell_j['d'] + 0.5 * ell_j['b'] * ell_j['e']
        E_j = 0.5 * ell_j['b'] * ell_j['e'] - ell_j['a'] * ell_j['e']
        F_j = -0.25 * ell_j['b']**2 + ell_j['a'] * ell_j['c']
        
        # Solve system A*x^2 + B*xy + C*y^2 + D*x + E*y + F = 0 for intersection (x_s, y_s)
        # The source does not provide the specific algebraic solver for conic intersection.
        # x_s, y_s = solve_conic_intersection(A_i, B_i, C_i, D_i, E_i, F_i, A_j, B_j, C_j, D_j, E_j, F_j)
        
        # Tangent line: x_s * x + y_s * y + 1 = 0
        # tangent_lines.append((x_s, y_s))
        
    # Step 4: Rendering
    # The source does not provide the exact implementation of the modified alpha-blending function.
    # return render_modified_alpha_blend(vertex_2ds, tangent_lines, primitive_type)
    pass
```

### Parameters and Edge Cases
*   **Threshold**: The boundary ellipse is defined at Gaussian value $1/255$. Points outside this contour are truncated to zero [S1, unofficial].
*   **Complexity**: Calculating the common tangent line between each pair of boundary ellipses has $O(1)$ time complexity [S1, unofficial].
*   **Pitfalls**: Numerical instability may occur if the covariance matrix is singular or near-singular. The source notes that avoiding numerical instability is a reason for the $1/255$ truncation [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The core geometric operation (common tangent calculation) is $O(1)$ per pair of vertices [S1, unofficial]. The overall complexity depends on the number of primitives and the cost of the modified alpha blending.
*   **Memory Complexity**: Requires storing additional parameters for non-elliptical primitives (relative 2D coordinates for triangles) compared to standard Gaussian ellipses [S1, unofficial].
*   **Strengths**: Enables accurate reconstruction of object surfaces using mixed primitives (Gaussian lines and triangles) [S1, unofficial].
*   **Limitations**: More complex rendering pipeline due to the need for re-sketching and dual ellipse calculations. Not directly compatible with standard Gaussian Splatting pipelines without modification [S1, unofficial].
*   **When to use**: Not given in the sources.

## Background
Compositional Splatting was introduced in the MP-GS (Mixed-Primitive-based Gaussian Splatting) framework [S1, unofficial]. It addresses the limitations of standard Gaussian Splatting, which uses only ellipsoidal primitives, by introducing Gaussian lines and triangles [S1, unofficial]. The method leverages the observation that line segments and triangles can be represented as compositions of their vertices, allowing for a compositional splatting approach [S1, unofficial].

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
