# Normal Field Supervision via Rendering and Depth Derivatives

Also called: Learnable Unit Normal with Depth Gradient Supervision, Lemma 2, Phys3DGS Normal Regularizer, Pseudo-Normal Supervision, Supervision of Normal Field

## Overview
This method supervises the learnable surface normal attributes of 3D Gaussians by enforcing consistency between the rendered normal map and the geometric normals derived from the rendered depth map. It solves the problem of noisy or inconsistent normal estimates in Gaussian Splatting by using the depth map as a geometry prior. It differs from pure gradient-based approaches by explicitly optimizing normal attributes per Gaussian and supervising them against pseudo-normals computed from expected depth [S1, unofficial][S2].

## How it works
**Inputs:**
- Rendered depth map $\hat{D}$ of resolution $W \times H$.
- Rendered normal map $\hat{N}$ of resolution $W \times H$, obtained by alpha-blending the learnable unit normals $n_i$ of the splatted Gaussians.

**Outputs:**
- Normal consistency loss $L_n$ used for gradient-based optimization of Gaussian parameters.

**Data Structures:**
- $\hat{D}$: 2D array of floats, where $\hat{D}(x,y)$ is the median depth at pixel $(x,y)$.
- $\hat{N}$: 2D array of 3D vectors, where $\hat{N}(x,y)$ is the alpha-blended normal at pixel $(x,y)$.
- $n_i$: 3D vector, the learnable unit normal attribute of the $i$-th Gaussian.

**Steps:**
1. **Render Normal Map:** Rasterize the 3D Gaussian Splatting representation. For each pixel, compute the alpha-blended normal $\hat{N}(x,y)$ using the learnable normals $n_i$ of the contributing Gaussians.
2. **Compute Pseudo-Normal:** Calculate the pseudo-ground truth normal $\tilde{N}(x,y)$ from the gradient of the rendered depth map $\hat{D}$ under a local planarity assumption [S1, unofficial].
   - Compute the image gradients of the depth map: $\nabla_x d$ and $\nabla_y d$. These are the differences in depth values between nearby pixels.
   - Compute the cross product of these gradients.
   - Normalize the resulting vector to get $\tilde{N}(x,y)$.
   $$ \tilde{N}(x, y) = \frac{\nabla_x d \times \nabla_y d}{|\nabla_x d \times \nabla_y d|} $$
   where $d$ are nearby depth points of pixel $(x, y)$ [S1, unofficial].
3. **Compute Loss:** Quantify the normal consistency as an L1 loss between the rendered normal $\hat{N}$ and the pseudo-normal $\tilde{N}$ [S1, unofficial]:
   $$ L_n = |\hat{N} - \tilde{N}| $$
   This loss encourages the rendered normal field to match the true normal of the surface at the 0.5-isosurface of the transmittance [S2].

**Parameters:**
- **Depth Metric:** The depth used for gradient computation is the median value of the free-flight distance $t^*_{o,w}$, not the expectancy, for robustness to outliers [S2].
- **Median Depth Definition:** The median depth $t$ is the infimum of $t$ such that transmittance $T_{o,w}(t) \leq 0.5$ [S2]. In point-based rendering, this is the depth $t^G_{o,w}$ of the first Gaussian along the ray for which the transmittance reaches the 0.5 threshold [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_loss(rendered_depth, rendered_normal):
    """
    Computes the normal consistency loss between rendered normals and 
    pseudo-normals derived from depth gradients.
    
    Args:
        rendered_depth (Tensor): Depth map of shape (H, W), float32.
        rendered_normal (Tensor): Normal map of shape (H, W, 3), float32.
        
    Returns:
        Tensor: Scalar loss value.
    """
    # Compute depth gradients using finite differences
    # Gradient in x direction: difference between adjacent columns
    depth_grad_x = rendered_depth[:, 1:] - rendered_depth[:, :-1]
    # Pad to maintain spatial dimensions
    depth_grad_x = torch.cat([depth_grad_x, depth_grad_x[:, -1:] - depth_grad_x[:, -1:]], dim=1)
    
    # Gradient in y direction: difference between adjacent rows
    depth_grad_y = rendered_depth[1:, :] - rendered_depth[:-1, :]
    # Pad to maintain spatial dimensions
    depth_grad_y = torch.cat([depth_grad_y, depth_grad_y[-1:, :] - depth_grad_y[-1:, :]], dim=0)
    
    # Compute pseudo-normal via cross product of depth gradients
    # Normal vector is perpendicular to the tangent plane defined by depth gradients
    pseudo_normal = torch.cross(depth_grad_x.unsqueeze(-1), depth_grad_y.unsqueeze(-1), dim=-1)
    
    # Normalize the pseudo-normal
    norm = torch.norm(pseudo_normal, dim=-1, keepdim=True)
    # Avoid division by zero
    norm = torch.clamp(norm, min=1e-8)
    pseudo_normal = pseudo_normal / norm
    
    # Compute L1 loss between rendered normal and pseudo-normal
    loss = torch.mean(torch.abs(rendered_normal - pseudo_normal))
    
    return loss
```

**Edge Cases and Pitfalls:**
- **Degenerate Gradients:** If the depth map is flat or noisy, the cross product may be near zero, leading to unstable normals. The normalization step includes a clamp to prevent division by zero [G].
- **Boundary Conditions:** Depth gradients at image boundaries require padding or special handling to maintain spatial alignment with the normal map [G].
- **Specular Surfaces:** Artifacts in the normal field may correspond to under-reconstructed parts of the scene, such as specular spots or sky regions, where depth estimates are unreliable [S2].

## Subtopics

### [Learnable Normal Estimation with Pseudo-Normal Supervision](10917-learnable-normal-estimation-with-pseudo-normal-supervision.md)
This method assigns a learnable unit normal to each 3D Gaussian to replace noisy gradient-based estimates, preventing the optimization from overfitting to training lighting conditions by aligning rendered normals with geometry priors derived from expected depth gradients. It differs from pure gradient-based approaches by explicitly optimizing normal attributes per Gaussian and supervising them against pseudo-normals computed from expected depth.

## Operating characteristics
- **Time Complexity:** The method adds a constant overhead per pixel for computing depth gradients and the cross product, which is $O(W \times H)$ per frame.
- **Memory Complexity:** Requires storing the rendered depth and normal maps, $O(W \times H)$ for depth and $O(W \times H \times 3)$ for normals.
- **Strengths:** Provides a robust supervision signal for normals without requiring explicit mesh reconstruction or additional geometric priors. The use of median depth makes it robust to outliers [S2].
- **Limitations:** Relies on the accuracy of the rendered depth map. In regions with specular reflections or transparent surfaces, depth estimates may be noisy, leading to inaccurate pseudo-normals [S2].
- **When to Use:** Suitable for scenes where consistent surface normals are required for downstream tasks like mesh extraction or lighting estimation, and where explicit normal supervision is not available.

## Background
The method originates from the work on high-fidelity surface reconstruction using oriented Gaussians [S2] and reflection-aware rendering in Gaussian Splatting [S1, unofficial]. It leverages the relationship between depth derivatives and surface normals, a concept well-established in computer vision [S2]. The specific formulation using median depth for robustness is detailed in [S2].

## Sources
[S1] Gaussian Splatting in Mirrors: Reflection-Aware Rendering via Virtual Camera Optimization, https://doi.org/10.48550/arxiv.2410.01614, fetched 2026-10-26 (unofficial)
[S2] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
