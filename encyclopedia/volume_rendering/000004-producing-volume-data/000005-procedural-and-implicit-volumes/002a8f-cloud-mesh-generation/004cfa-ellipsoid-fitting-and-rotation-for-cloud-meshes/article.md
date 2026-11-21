# Ellipsoid Fitting and Rotation for Cloud Meshes

Also called: Cloud Generation from 3D Meshes, Mesh Decimation and Ellipsoid Smoothing, Mesh-to-Ellipsoid conversion, Rodrigues' Rotation Formula for Cloud Geometry, Triangle-based ellipsoid cloud generation

## Overview
This method converts high-resolution 3D meshes (e.g., from Blender or Maya) into volumetric cloud geometry by replacing triangle faces with fitted ellipsoids [S1]. It solves the problem of rendering complex, organic shapes in real-time by decimating the input mesh and approximating its surface with smooth primitives [S1]. Unlike raw mesh rendering, this approach acts as a real-time filter that produces softer, cumuliform shapes suitable for smoke and cloud simulation [S1]. It builds on [Cloud Mesh Generation](4822-cloud-mesh-generation.md) by providing the specific geometric fitting and optimization steps required to transform discrete triangles into continuous volumetric primitives.

## How it works
**Inputs:** A decimated 3D mesh consisting of $N$ triangles, where each triangle has vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$.
**Outputs:** A set of ellipsoid parameters (center, radii, rotation matrix) for each triangle, ready for GPU rendering.

**Data Structures:**
*   **Triangle:** Struct containing three 3D vertex coordinates $\mathbf{v}_i \in \mathbb{R}^3$.
*   **Ellipsoid:** Struct containing:
    *   Center $\mathbf{c} \in \mathbb{R}^3$
    *   Radii $(a, b, c) \in \mathbb{R}^3$
    *   Rotation Matrix $\mathbf{R} \in \mathbb{R}^{3 \times 3}$

**Steps:**

1.  **Mesh Decimation:** Reduce the input mesh to 300–700 triangles (or $\le 1000$) to ensure real-time performance [S1]. A decimation rate of 80–95% is acceptable [S1].
2.  **Barycenter Calculation:** For each triangle, compute the barycenter $\mathbf{B}$ [S1]:
    $$ \mathbf{B} = \frac{\mathbf{v}_1 + \mathbf{v}_2 + \mathbf{v}_3}{3} \quad \text{(Eq. 22)} $$
3.  **Scaling:** Scale the triangle vertices relative to the barycenter using a scale factor $s$ [S1]. The factor $s$ typically falls in the range $(0.1, 2]$ [S1]. The scaled vertices $\mathbf{v}'_i$ are calculated as:
    $$ \mathbf{v}'_i = \mathbf{B} + s(\mathbf{v}_i - \mathbf{B}) \quad \text{(Eq. 23, 24)} $$
4.  **Radii Determination:** Calculate the maximum distance from the barycenter to any scaled vertex. This distance defines the maximum radius of the circumscribing ellipsoid. The radii $(a, b, c)$ are derived from this maximum extent [S1].
5.  **Rotation Optimization:** Align the ellipsoid's maximum radius with the direction of the maximum distance from the barycenter to the triangle vertices to produce a softer shape [S1]. This uses Rodrigues' Rotation Formula [S1]:
    *   **Axis and Angle:** Calculate the rotation axis $\mathbf{k}$ and angle $\theta$ using cross and dot products of the relevant direction vectors [S1]:
        $$ \mathbf{k} = \frac{\mathbf{u} \times \mathbf{v}}{\|\mathbf{u} \times \mathbf{v}\|}, \quad \theta = \arccos\left(\frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\| \|\mathbf{v}\|}\right) \quad \text{(Eq. 27, 28)} $$
        *(Note: $\mathbf{u}$ and $\mathbf{v}$ represent the initial and target direction vectors for alignment.)*
    *   **Skew-Symmetric Matrix:** Compute matrix $\mathbf{A}$ corresponding to the axis vector $\mathbf{x}$ (where $\mathbf{x}$ is the normalized rotation axis) [S1]:
        $$ \mathbf{A} = \begin{bmatrix} 0 & -x_z & x_y \\ x_z & 0 & -x_x \\ -x_y & x_x & 0 \end{bmatrix} \quad \text{(Eq. 30)} $$
    *   **Rotation Matrix:** Calculate $\mathbf{R}$ using the exponential map (Rodrigues' formula) [S1]:
        $$ \mathbf{R} = \mathbf{I} + (\sin \theta)\mathbf{A} + (1 - \cos \theta)\mathbf{A}^2 \quad \text{(Eq. 29)} $$
6.  **GPU Transfer:** Pre-calculate formulas (1)–(3) on the CPU. Only the final rotation matrix $\mathbf{R}$ and ellipsoid parameters are passed to the GPU for rendering [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_barycenter(v1, v2, v3):
    """Calculate barycenter B of a triangle."""
    return (v1 + v2 + v3) / 3.0

def scale_triangle(v1, v2, v3, barycenter, scale_factor):
    """Scale triangle vertices from barycenter by scale_factor."""
    v1_scaled = barycenter + scale_factor * (v1 - barycenter)
    v2_scaled = barycenter + scale_factor * (v2 - barycenter)
    v3_scaled = barycenter + scale_factor * (v3 - barycenter)
    return v1_scaled, v2_scaled, v3_scaled

def get_max_radius(barycenter, v1, v2, v3):
    """Calculate maximum distance from barycenter to vertices."""
    d1 = np.linalg.norm(v1 - barycenter)
    d2 = np.linalg.norm(v2 - barycenter)
    d3 = np.linalg.norm(v3 - barycenter)
    return max(d1, d2, d3)

def compute_rotation_matrix(u, v):
    """
    Compute rotation matrix R to align vector u with vector v.
    Uses Rodrigues' Rotation Formula.
    """
    # Normalize vectors
    u_norm = u / np.linalg.norm(u)
    v_norm = v / np.linalg.norm(v)
    
    # Calculate axis and angle
    cross_product = np.cross(u_norm, v_norm)
    dot_product = np.dot(u_norm, v_norm)
    
    # Handle parallel vectors
    if np.linalg.norm(cross_product) < 1e-6:
        if dot_product > 0:
            return np.eye(3) # No rotation needed
        else:
            # 180 degree rotation, need an arbitrary orthogonal axis
            # [G] Fallback: rotate around an orthogonal vector
            axis = np.array([1, 0, 0])
            if abs(np.dot(axis, u_norm)) > 0.9:
                axis = np.array([0, 1, 0])
            axis = axis - np.dot(axis, u_norm) * u_norm
            axis = axis / np.linalg.norm(axis)
            theta = math.pi
        cross_product = axis
        theta = math.pi
    else:
        cross_product = cross_product / np.linalg.norm(cross_product)
        theta = math.acos(np.clip(dot_product, -1.0, 1.0))

    # Skew-symmetric matrix A
    x = cross_product
    A = np.array([
        [0, -x[2], x[1]],
        [x[2], 0, -x[0]],
        [-x[1], x[0], 0]
    ])
    
    # Rotation matrix R = I + sin(theta)A + (1-cos(theta))A^2
    R = np.eye(3) + math.sin(theta) * A + (1 - math.cos(theta)) * np.dot(A, A)
    return R

def fit_ellipsoid_to_triangle(v1, v2, v3, scale_factor=1.0):
    """
    Main function to fit an ellipsoid to a triangle.
    Returns center, radii, and rotation matrix.
    """
    # 1. Barycenter
    B = compute_barycenter(v1, v2, v3)
    
    # 2. Scale vertices
    v1_s, v2_s, v3_s = scale_triangle(v1, v2, v3, B, scale_factor)
    
    # 3. Determine radii (max distance defines the primary radius)
    max_r = get_max_radius(B, v1_s, v2_s, v3_s)
    
    # [G] The source defines radii (a,b,c) based on max distance but does not specify 
    # how minor axes are derived. We assume a sphere or isotropic scaling for simplicity 
    # if not specified, or use the max_r for the largest axis.
    # The source states: "maximum distance ... is used to calculate a circumscribing ellipsoid with radii (a, b, c)"
    # We set the largest radius to max_r. Other radii are not explicitly defined in the text.
    # [G] Assuming isotropic ellipsoid (sphere) for the base fit, or that a=b=c=max_r 
    # is a simplification of "circumscribing". However, the rotation implies anisotropy.
    # Since specific minor axis calculation is missing, we return max_r for all or note it.
    # Given the rotation optimization aligns the "maximum radius", we assume the ellipsoid 
    # has one major axis aligned with the max distance vector.
    
    # 4. Rotation Optimization
    # Find the vertex with the maximum distance to align the ellipsoid
    dists = [
        np.linalg.norm(v1_s - B),
        np.linalg.norm(v2_s - B),
        np.linalg.norm(v3_s - B)
    ]
    max_idx = np.argmax(dists)
    max_vertex = [v1_s, v2_s, v3_s][max_idx]
    
    # Direction vector from barycenter to the furthest vertex
    target_dir = max_vertex - B
    
    # Initial direction (e.g., local Z axis or arbitrary reference)
    # [G] The source says "align the maximum radius ... with the maximum distance".
    # It implies rotating the ellipsoid's principal axis to match this vector.
    # We assume the ellipsoid's initial principal axis is along Z (0,0,1).
    initial_axis = np.array([0.0, 0.0, 1.0])
    
    # Compute rotation matrix to align initial_axis with target_dir
    R = compute_rotation_matrix(initial_axis, target_dir)
    
    return {
        'center': B,
        'radii': [max_r, max_r, max_r], # [G] Specific minor radii not defined in source
        'rotation_matrix': R
    }
```

**Parameters:**
*   **Scale Factor ($s$):** Typically in the range $(0.1, 2]$ [S1].
*   **Mesh Resolution:** 300–700 triangles after decimation [S1]. Up to 1000 triangles is the upper limit for standard GPUs [S1].

**Edge Cases and Pitfalls:**
*   **Performance:** Without pre-calculation on the CPU and decimation, performance is hindered on lower-speed GPUs [S1].
*   **Precision:** The rotation calculation relies on stable cross/dot products; parallel vectors require special handling [G].
*   **Boundary:** The method assumes the input mesh is already decimated. Fitting to high-poly meshes directly is not supported [S1].

## Operating characteristics
*   **Time Complexity:** Pre-calculation is $O(N)$ where $N$ is the number of triangles. This is done once on the CPU. GPU rendering is $O(N)$ per frame.
*   **Memory Complexity:** $O(N)$ to store ellipsoid parameters (center, radii, rotation matrix) for each triangle.
*   **Performance:** Significantly improves performance on lower-end GPUs by reducing vertex count and simplifying geometry to ellipsoids [S1]. Benchmarks confirm no hindrance on lower-speed hardware [S1].
*   **Strengths:** Real-time capability; produces soft, organic cloud shapes; leverages standard 3D modeling tools [S1].
*   **Limitations:** Requires manual or automated decimation; approximation error increases with high decimation rates (>95%) though visual quality remains acceptable [S1].
*   **When to use:** Real-time cloud rendering where organic shapes are needed and high-fidelity mesh rendering is too expensive.
*   **When not to use:** Static, high-precision cloud visualization where decimation artifacts are visible.

## Background
This method was introduced in the context of real-time GPU volumetric cloud rendering, specifically for generating "suggestive cumuliforms" [S1]. It allows artists to use standard 3D modeling software (Blender, Maya) to create cloud shapes, which are then converted into volumetric primitives for rendering [S1]. The technique is part of a broader set of geometry innovations for real-time cloud rendering [S1].

## Sources
[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
