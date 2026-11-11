# Normal Field Supervision via Rendering and Depth Derivatives

Also called: Lemma 2, Supervision of Normal Field

## Overview
This method supervises the learnable surface normal field of a 3D Gaussian Splatting representation by enforcing geometric consistency between the rendered normal maps and the spatial derivatives of the rendered depth map [S1]. It addresses the problem of noisy or inconsistent normals in point-based rendering by leveraging the fact that the gradient of the depth image corresponds to the surface normal of the expected visible surface [S1]. Unlike [Learnable Normals Parameterization](9615-learnable-normals-parameterization.md), which focuses on parameterization stability, this method provides an external geometric constraint derived from the scene's geometry rather than internal Gaussian attributes.

## How it works
The method operates by comparing two independent estimates of the surface normal for each pixel and minimizing the discrepancy between them.

**Inputs:**
*   3D Gaussian Splatting representation with learnable normals $\mathbf{n}_i$ for each Gaussian $i$.
*   Camera parameters (origin $\mathbf{o}$, direction $\mathbf{w}$).
*   Depth rendering capability.
*   Normal rendering capability.

**Outputs:**
*   Gradient of the loss function with respect to the learnable normals $\mathbf{n}_i$.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores a learnable unit normal vector $\mathbf{n}_i \in \mathbb{R}^3$.
*   **Rendered Buffers:**
    *   Normal Map $\mathbf{N}_{rendered}(u,v) \in \mathbb{R}^3$: Alpha-blended normals.
    *   Depth Map $D(u,v) \in \mathbb{R}$: Median depth values.

**Steps:**

1.  **Render Normal Map:**
    Rasterize the 3D Gaussian Splatting representation using alpha-blending to produce a normal map. For a ray $\mathbf{r}_{o,w}$, the rendered normal $\mathbf{N}_{rendered}$ is the weighted average of the Gaussian normals $\mathbf{n}_i$ along the ray, weighted by their contribution to the color/opacity [S1].

2.  **Render Median Depth Map:**
    Compute the depth map $D(u,v)$ using the median free-flight distance $t^*_{o,w}$ rather than the expectancy. The median depth $t$ is defined as the infimum of $t$ such that the transmittance $T_{o,w}(t) \leq 0.5$ [S1].
    
    In point-based rendering, this is implemented as the depth $t^G_{o,w}$ of the first Gaussian along the ray where the accumulated transmittance drops below 0.5 [S1].
    
    $$
    \text{Median}_M(t_{o,w}^*) = \inf \{ t \in [0, +\infty) : T_{o,w}(t) \leq 0.5 \}
    $$
    
    This choice is made because the median is more robust to statistical outliers than the expectancy (mean) [S1].

3.  **Compute Depth-Derived Normals:**
    Estimate the surface normal $\mathbf{N}_{depth}$ by computing the image-space derivatives of the rendered depth map $D(u,v)$ [S1]. According to [39], rendering depth estimates the expectancy of the free-flight distance, and its derivative yields the normal of the expected visible surface [S1].
    
    Let $D_u$ and $D_v$ be the partial derivatives of the depth map with respect to pixel coordinates $u$ and $v$. The normal is derived from the cross product of the tangent vectors in the image plane projected into 3D space. Specifically, if $\mathbf{p}(u,v)$ is the 3D point at depth $D(u,v)$, the normal is proportional to $\frac{\partial \mathbf{p}}{\partial u} \times \frac{\partial \mathbf{p}}{\partial v}$ [S1].

4.  **Supervise Normals:**
    Minimize the difference between the rendered normal map $\mathbf{N}_{rendered}$ and the depth-derived normal $\mathbf{N}_{depth}$ [S1]. This supervision promotes the value of the normal field at the 0.5-isosurface of the transmittance to match the true surface normal [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_supervision_loss(gaussians, camera, depth_map):
    """
    Computes the loss for normal field supervision.
    
    Args:
        gaussians: List of Gaussian objects, each with attribute 'normal' (n_i).
        camera: Camera object with 'origin' (o) and 'direction' (w) for each pixel.
        depth_map: 2D array of median depth values D(u,v).
        
    Returns:
        loss: Scalar loss value.
        grad_normals: Gradients for each Gaussian's normal attribute.
    """
    # Step 1: Render Normal Map
    # Alpha-blend the learnable normals n_i along each ray
    # N_rendered[u,v] = sum(alpha_i * n_i) / sum(alpha_i) for all Gaussians i on ray (u,v)
    N_rendered = render_normal_map(gaussians, camera)
    
    # Step 2: Depth Map is already provided as median depth
    # D = depth_map
    
    # Step 3: Compute Depth-Derived Normals
    # Calculate spatial gradients of the depth map
    D_u = gradient_x(depth_map)
    D_v = gradient_y(depth_map)
    
    # Convert depth gradients to 3D normals
    # The normal is derived from the cross product of the surface tangents
    # Tangent_u = camera.jacobian_u * D_u + camera.tangent_u
    # Tangent_v = camera.jacobian_v * D_v + camera.tangent_v
    # N_depth = normalize(Tangent_u x Tangent_v)
    N_depth = depth_gradients_to_normals(depth_map, D_u, D_v, camera)
    
    # Step 4: Compute Loss
    # Enforce consistency between rendered normals and depth-derived normals
    # Loss = mean(1 - dot(N_rendered, N_depth)) or similar angular distance
    loss = mean_cosine_distance(N_rendered, N_depth)
    
    # Backpropagation computes gradients for n_i via the rendering process
    grad_normals = backpropagate(loss, gaussians)
    
    return loss, grad_normals
```

**Edge Cases and Pitfalls:**
*   **Specular Surfaces:** Artifacts in the normal field correspond to under-reconstructed parts of the scene, such as specular spots (e.g., on a kitchen table) or reflective surfaces (e.g., a TV) [S1].
*   **Background/Sky:** Regions with no defined surface geometry, such as the sky in outdoor scenes (e.g., Courthouse scene), may produce inconsistent normals [S1].
*   **Median vs. Mean:** Using the expectancy (mean) depth instead of the median depth makes the supervision sensitive to statistical outliers, leading to less robust normal estimation [S1].

## Subtopics

## Operating characteristics
*   **Complexity:** The method adds the cost of rendering a normal map and computing spatial derivatives of the depth map. The derivative computation is $O(N)$ where $N$ is the number of pixels.
*   **Robustness:** The use of median depth provides robustness to outliers compared to mean depth [S1].
*   **Limitations:** The method relies on the depth map being a good estimate of the surface. In areas with transparency, specularities, or missing geometry, the depth derivative may not correspond to the true surface normal, leading to artifacts [S1].

## Background
This method is introduced in "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1]. It is formally stated as Lemma 2 (Supervision of Normal Field) in the source paper. The approach leverages the relationship between depth derivatives and surface normals, a concept previously utilized in [39] (cited in [S1]), to provide geometric supervision for the oriented Gaussian representation.

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
