# GRay

Also called: GRay: Ray Tracing 3D Gaussians Near the Speed of Splats

## Overview
GRay is a fast ray-tracing method for 3D Gaussian Splatting (3DGS) designed to close the performance gap between ray tracing and rasterization by leveraging the algorithmic advantage of ray tracing: it evaluates only intersected Gaussians, allowing logarithmic scaling in dense scenes with tiny Gaussians [S1]. Unlike 3DGS rasterization which operates on tiles and evaluates all Gaussians overlapping a tile for every pixel (linear scaling), GRay uses a Bounding Volume Hierarchy (BVH) to select and evaluate exactly the set of Gaussians intersecting each pixel [S1]. It builds on [3D Gaussian Ray Tracing](8101-3d-gaussian-ray-tracing.md), [EPBRR](8102-epbrr.md), and [EDGS](8103-edgs.md) [S1].

## How it works
**Inputs:** Camera parameters (position, orientation, intrinsics), a set of 3D Gaussians defined by mean $\mu$, covariance $\Sigma$ (or rotation $R$ and scale $S$), and color/opacity attributes.
**Outputs:** Rendered image with accumulated color and opacity per pixel.

**Data Structures:**
1.  **3D Gaussians:** Each Gaussian $i$ is defined by:
    *   Mean position $\mu_i \in \mathbb{R}^3$.
    *   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, derived from rotation $R_i \in SO(3)$ and scale $S_i \in \mathbb{R}^3$ such that $\Sigma_i = R_i S_i S_i^T R_i^T$.
    *   Color $c_i \in \mathbb{R}^3$ (or spherical harmonics coefficients).
    *   Opacity $\alpha_i \in [0, 1]$.
2.  **Bounding Volume Hierarchy (BVH):** A tree structure where each node contains an Oriented Bounding Box (OBB) enclosing its children. Leaf nodes contain individual Gaussians. The OBB is defined by a center $c_{node}$, three orthogonal axes $u, v, w$, and extents $e_x, e_y, e_z$.
3.  **Per-pixel Linked Lists:** Used to store the list of intersected Gaussians for each pixel during the accumulation phase, relieving the need for fixed-size arrays [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the Gaussian set. GRay uses a dense initialization strategy (based on EDGS) which creates many small Gaussians [S1]. This slows down rasterization but speeds up ray tracing [S1].
2.  **BVH Construction:** Build a BVH over the 3D Gaussians using Oriented Bounding Boxes (OBBs). Unlike methods using Axis-Aligned Bounding Boxes (AABBs) that require anisotropy regularization, GRay uses OBBs and does not require anisotropy regularization [S1].
3.  **Ray Generation:** For each pixel $(u, v)$ in the image, generate a ray $r(t) = o + td$ using the camera parameters.
4.  **Ray-BVH Traversal:** Traverse the BVH to find all Gaussians intersected by the ray.
    *   Start at the root node.
    *   For each node, check if the ray intersects the node's OBB.
    *   If intersected, traverse children.
    *   At leaf nodes, check for intersection with the contained Gaussian.
    *   Collect all intersected Gaussians.
5.  **Sorting:** Sort the intersected Gaussians by their distance from the camera origin along the ray.
6.  **Accumulation (Detached Hybrid Transparency):** Render the Gaussians in sorted order using alpha compositing.
    *   Initialize transmittance $T = 1.0$ and accumulated color $C = 0.0$.
    *   For each Gaussian $i$ in sorted order:
        *   Project the 3D Gaussian to 2D to compute its screen-space covariance $\Sigma'$.
        *   Compute the 2D Gaussian's contribution to the pixel.
        *   Update transmittance: $T \leftarrow T \cdot (1 - \alpha_i)$.
        *   Update color: $C \leftarrow C + c_i \cdot \alpha_i \cdot T$.
        *   If $T < \epsilon$ (transmittance threshold), terminate early.
    *   The "detached hybrid transparency" technique follows the principle of RaySplats but excludes the fixed cap on the number of Gaussians and the secondary threshold, relying on per-pixel linked lists instead [S1].
7.  **Output:** Write $C$ to the pixel buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gray(cameras, gaussians, bvh, image_width, image_height):
    # Initialize output buffers
    color_buffer = zeros((image_height, image_width, 3))
    
    # Pre-compute camera rays
    rays = generate_rays(cameras, image_width, image_height)
    
    for y in range(image_height):
        for x in range(image_width):
            ray = rays[y, x]
            
            # 1. Ray-BVH Traversal
            intersected_gaussians = []
            traverse_bvh(bvh, ray, intersected_gaussians)
            
            # 2. Sort by distance
            intersected_gaussians.sort(key=lambda g: distance(ray.origin, g.mean))
            
            # 3. Accumulate
            T = 1.0
            C = [0.0, 0.0, 0.0]
            epsilon = 1e-4
            
            for g in intersected_gaussians:
                # Project 3D Gaussian to 2D
                sigma_2d = project_gaussian(g, ray, cameras)
                
                # Compute 2D Gaussian value at pixel center
                # This involves computing the Mahalanobis distance
                dist_sq = mahalanobis_dist([x, y], g.screen_mean, sigma_2d)
                alpha = g.opacity * exp(-0.5 * dist_sq)
                
                # Clamp alpha
                alpha = clamp(alpha, 0.0, 1.0)
                
                # Update color and transmittance
                C = C + g.color * alpha * T
                T = T * (1.0 - alpha)
                
                # Early termination
                if T < epsilon:
                    break
            
            color_buffer[y, x] = C
            
    return color_buffer

def traverse_bvh(node, ray, results):
    if not ray_intersects_obb(ray, node.obb):
        return
    
    if node.is_leaf:
        for gaussian in node.gaussians:
            if ray_intersects_gaussian(ray, gaussian):
                results.append(gaussian)
    else:
        for child in node.children:
            traverse_bvh(child, ray, results)
```

**Parameters:**
*   Transmittance threshold $\epsilon$: Not given in the sources.
*   Number of iterations: 15,000 [S1].
*   Initial Gaussians: 3.27M [S1].
*   Final Gaussians: 1.52M [S1].

**Edge Cases and Pitfalls:**
*   **Dense Scenes:** GRay is designed to handle dense scenes with tiny Gaussians better than rasterization [S1].
*   **Anisotropy:** GRay uses OBBs, avoiding the need for anisotropy regularization required by AABB-based methods [S1].
*   **Memory:** Uses per-pixel linked lists, avoiding fixed-size array limits [S1].

## Operating characteristics
*   **Speed:** GRay achieves 248 FPS on the 13 standard 3DGS evaluation scenes, compared to 3DGRT's 68 FPS [S1]. It renders nearly 4× faster than 3DGRT [S1].
*   **Optimization Time:** GRay optimization time is 05:40, compared to 3DGRT's 55:01 and 3DGS's 06:18 [S1]. It optimizes nearly 10× faster than 3DGRT [S1].
*   **Quality:** GRay achieves 26.47 PSNR, compared to 3DGRT's 26.77 PSNR and 3DGS's 27.10 PSNR [S1]. It has competitive speed with 3DGS albeit at somewhat lower quality [S1].
*   **Scaling:** Ray tracing evaluates only intersected Gaussians, yielding potentially logarithmic scaling in the number of primitives if they are sufficiently small [S1]. Dense initialization speeds up ray tracing but slows down rasterization [S1].
*   **Strengths:** Fast rendering and optimization for dense scenes with small Gaussians. No anisotropy regularization needed.
*   **Limitations:** Slightly lower quality than 3DGS and 3DGRT.
*   **When to use:** When rendering speed and optimization time are critical, and the scene contains many small Gaussians.
*   **When not to use:** When maximum reconstruction quality is required, or when the scene is sparse and rasterization is sufficient.

## Background
GRay was introduced by Yohan Poirier-Ginter, Jean-François Lalonde, and George Drettakis in 2026 [S1]. It is designed to address the performance gap between ray tracing and rasterization in 3D Gaussian Splatting [S1]. It builds on previous work: 3D Gaussian Ray Tracing (3DGRT) [S1], EPBRR [S1], and EDGS [S1]. The method leverages the algorithmic difference between rasterization and ray tracing, where ray tracing evaluates only intersected Gaussians [S1].

## Sources
[S1] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
