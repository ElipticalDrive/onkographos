# Walking on Triangle


## Overview
Walking on Triangle is a constraint mechanism that binds 3D Gaussians to the surface of a triangle mesh, ensuring they remain within the boundaries of their corresponding triangles during optimization and deformation [S1, unofficial][S2]. It solves the problem of Gaussians drifting off-mesh or piling up at triangle edges when the underlying mesh is manipulated, animated, or deformed [S1, unofficial]. Unlike methods that rely heavily on precise mesh accuracy, it defines a local triangle space to ensure robust results even with inaccurate meshes [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh (vertices, faces), 3D Gaussian attributes (position, scale, rotation, opacity, color), and barycentric coordinates $(u, v)$ for each Gaussian relative to its host triangle.
*   **Output:** Updated Gaussian positions and attributes that respect the mesh topology and local triangle constraints.

**Data Structures:**
*   **Gaussian State:** Each Gaussian $i$ is associated with a specific triangle $T_k$ of the mesh.
*   **Barycentric Coordinates:** For each Gaussian, two scalar values $u_i, v_i$ representing its position within triangle $T_k$. The third barycentric coordinate is $w_i = 1 - u_i - v_i$.
*   **Local Triangle Space:** Defined by the three vertices of triangle $T_k$: $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$.

**Steps:**
1.  **Bind Gaussians to Mesh:** Assign each Gaussian to a specific triangle on the mesh. This association is typically established during initialization or via nearest-neighbor projection [S2].
2.  **Define Local Triangle Space:** For each triangle $T_k$, define the local coordinate system using its vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ [S2].
3.  **Optimize Attributes in Local Space:** During training, optimize Gaussian attributes (such as position offsets, scales, or rotations) within the context of the local triangle space [S2]. This ensures that deformations applied to the mesh are directly transferred to the Gaussians [S2].
4.  **Clip UV Values:** To prevent Gaussians from moving beyond the boundaries of their assigned triangle, clip the barycentric coordinates $u_i$ and $v_i$ [S1, unofficial]. The constraints are:
    $$ u_i \ge 0, \quad v_i \ge 0, \quad u_i + v_i \le 1 $$
    If a Gaussian's coordinates violate these bounds, they are projected back onto the nearest valid point within the triangle [S1, unofficial].
5.  **Transfer Manipulation:** When the mesh is deformed or manipulated, the new positions of the triangle vertices are used to recompute the 3D position of each Gaussian based on its fixed barycentric coordinates [S2]. This allows for large deformations, soft body simulation, and local manipulation [S2].

**Pitfalls:**
*   **Boundary Artifacts:** Disabling the walking-on-triangle mechanism causes Gaussians to stick and pile up on the boundaries of mesh triangles, leading to visible artifacts when the avatar is animated by novel poses [S1, unofficial].
*   **Mesh Accuracy:** While the method is robust to inaccurate meshes, extremely poor mesh topology may still lead to suboptimal binding [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clip_barycentric(u, v):
    """
    Clips barycentric coordinates to ensure they remain within the triangle.
    Args:
        u: float, first barycentric coordinate
        v: float, second barycentric coordinate
    Returns:
        tuple: (clipped_u, clipped_v)
    """
    # Ensure u >= 0
    if u < 0:
        u = 0.0
    # Ensure v >= 0
    if v < 0:
        v = 0.0
    # Ensure u + v <= 1
    if u + v > 1.0:
        # Project back to the edge u + v = 1
        # Normalize u and v to sum to 1, then scale to fit the constraint
        total = u + v
        u = u / total
        v = v / total
    return u, v

def update_gaussian_position(gaussian, triangle_vertices):
    """
    Updates the 3D position of a Gaussian based on its barycentric coordinates
    and the current positions of the triangle vertices.
    Args:
        gaussian: object with attributes u, v
        triangle_vertices: list of 3 vectors [v1, v2, v3]
    Returns:
        vector: new 3D position
    """
    v1, v2, v3 = triangle_vertices
    u, v = gaussian.u, gaussian.v
    # Clip coordinates to stay within triangle bounds
    u, v = clip_barycentric(u, v)
    # Compute position using barycentric interpolation
    # w = 1 - u - v
    pos = (1 - u - v) * v1 + u * v2 + v * v3
    return pos
```

## Subtopics

### Mesh-Embedded Gaussian Initialization and Deformation
The parent method that anchors Gaussians to a mesh via barycentric coordinates and normal displacement. Walking on Triangle is a specific constraint mechanism within this framework to prevent boundary artifacts. [Link](9559-mesh-embedded-gaussian-initialization-and-deformation.md)

### Mani-GS
A method that uses Walking on Triangle to enable large deformation and soft body simulation. It produces fewer artifacts and reduces blurring compared to SuGaR. [Link](9559-mesh-embedded-gaussian-initialization-and-deformation.md)

## Operating characteristics
*   **Time Complexity:** The clipping operation is $O(1)$ per Gaussian per frame. The position update is also $O(1)$. The overall complexity is dominated by the rendering and optimization steps of the 3D Gaussian Splatting pipeline.
*   **Memory Complexity:** Adds negligible memory overhead, storing only the barycentric coordinates $(u, v)$ for each Gaussian.
*   **Performance:**
    *   For the 'bala' head avatar, enabling Walking on Triangle increases PSNR from 29.91 to 30.04, SSIM from 0.933 to 0.938, and decreases LPIPS from 0.070 to 0.062 [S1, unofficial].
    *   For the 'male-3-casual' full-body avatar, it increases PSNR from 32.48 to 33.01, SSIM from 0.979 to 0.982, and decreases LPIPS from 0.024 to 0.020 [S1, unofficial].
*   **Strengths:** Prevents artifacts during animation, robust to mesh inaccuracies, enables direct mesh manipulation.
*   **Limitations:** Requires a valid triangle mesh and initial binding of Gaussians to triangles.

## Background
Walking on Triangle was introduced in the context of SplattingAvatar, a method for relightable modeling with explicit shape representations [S1, unofficial]. It was designed to address the performance compromises of previous explicit shape methods, particularly in maintaining pixel-wise metrics during animation [S1, unofficial]. The method is also highlighted in surveys of 3D editing based on NeRF and 3DGS as a key technique for mesh-based deformation [S2].

## Sources
[S1] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
