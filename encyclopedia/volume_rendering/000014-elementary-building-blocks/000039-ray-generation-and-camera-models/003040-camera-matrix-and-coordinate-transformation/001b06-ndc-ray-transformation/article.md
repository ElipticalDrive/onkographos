# NDC Ray Transformation

Also called: NDC Ray Conversion, NDC Ray Conversion for Forward-Facing Scenes, NDC Rays, Non-Deterministic Coordinates (NDC) Ray Transformation, Non-Dominated Coordinates (NDC) Ray Transformation, Normalized Device Coordinates Ray Transform, Unbounded Scene Ray Transformation

## Overview
The NDC Ray Transformation maps camera-space rays into a canonical bounding cube $[-1, 1]^3$ to enable uniform sampling in disparity (inverse depth) for forward-facing, unbounded scenes [S1][S3, S4, S5, unofficial]. Unlike standard ray generation which assumes bounded metric depth, this method shifts ray origins to the near plane and projects them such that linear sampling in the transformed space corresponds to linear sampling in disparity from the near plane to infinity [S1][S3, S4, unofficial]. It is primarily used for real-world captures (e.g., LLFF) where the scene extends to infinity, whereas standard ray generation is reserved for synthetic bounded scenes (e.g., Blender) [S6, unofficial].

## How it works
**Inputs:**
*   Ray origins $\mathbf{o} \in \mathbb{R}^3$ and directions $\mathbf{d} \in \mathbb{R}^3$ in camera space. The camera looks down the $-Z$ axis.
*   Camera intrinsics: focal length $f_{cam}$, image width $W$, image height $H$.
*   Near plane distance $n$ (scalar).
*   Far plane distance $f$ (scalar, typically $\infty$ for forward-facing scenes).

**Outputs:**
*   Transformed ray origins $\mathbf{o}' \in \mathbb{R}^3$ and directions $\mathbf{d}' \in \mathbb{R}^3$ in NDC space.

**Data Structures:**
*   Rays are stored as tensors of shape $(N, 3)$ for origins and directions, where $N$ is the number of rays.

**Steps:**

1.  **Shift Origin to Near Plane:**
    Calculate the parameter $t_n$ to move the ray origin $\mathbf{o}$ to the intersection with the near plane $z = -n$.
    $$ t_n = -\frac{n + o_z}{d_z} $$
    Update the origin:
    $$ \mathbf{o}_{shifted} = \mathbf{o} + t_n \mathbf{d} $$
    *Note: This step ensures that $t'=0$ in NDC space corresponds to the near plane in world space, allowing linear sampling of $t' \in [0, 1]$ to cover disparity from $n$ to $\infty$ [S1].*

2.  **Compute Projection Constants:**
    Define the scaling factors based on the pinhole camera model.
    $$ a_x = -\frac{f_{cam}}{W/2} $$
    $$ a_y = -\frac{f_{cam}}{H/2} $$
    For unbounded scenes (far plane $f \to \infty$), the Z-constants simplify to:
    $$ a_z = 1 $$
    $$ b_z = 2n $$
    *If a finite far plane $f$ is required, use the standard perspective projection constants:*
    $$ a_z = \frac{f+n}{f-n} $$
    $$ b_z = \frac{2fn}{f-n} $$

3.  **Project Origin to NDC:**
    Compute the NDC origin $\mathbf{o}'$ using the shifted origin $\mathbf{o}_{shifted}$ (denoted as $\mathbf{o}$ in the final formulas below for brevity, assuming the shift has occurred).
    $$ o'_x = a_x \frac{o_x}{o_z} $$
    $$ o'_y = a_y \frac{o_y}{o_z} $$
    $$ o'_z = a_z + \frac{b_z}{o_z} $$
    Substituting the simplified constants for unbounded scenes:
    $$ o'_x = -\frac{f_{cam}}{W/2} \frac{o_x}{o_z} $$
    $$ o'_y = -\frac{f_{cam}}{H/2} \frac{o_y}{o_z} $$
    $$ o'_z = 1 + \frac{2n}{o_z} $$

4.  **Project Direction to NDC:**
    Compute the NDC direction $\mathbf{d}'$.
    $$ d'_x = a_x \left( \frac{d_x}{d_z} - \frac{o_x}{o_z} \right) $$
    $$ d'_y = a_y \left( \frac{d_y}{d_z} - \frac{o_y}{o_z} \right) $$
    $$ d'_z = -\frac{b_z}{o_z} $$
    Substituting the simplified constants for unbounded scenes:
    $$ d'_x = -\frac{f_{cam}}{W/2} \left( \frac{d_x}{d_z} - \frac{o_x}{o_z} \right) $$
    $$ d'_y = -\frac{f_{cam}}{H/2} \left( \frac{d_y}{d_z} - \frac{o_y}{o_z} \right) $$
    $$ d'_z = -\frac{2n}{o_z} $$

5.  **Reparameterization Mapping:**
    The parameter $t'$ in NDC space is related to the original parameter $t$ by:
    $$ t' = \frac{t d_z}{o_z + t d_z} $$
    This maps $t=0$ to $t'=0$ and $t \to \infty$ to $t'=1$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def ndc_rays(o, d, focal, W, H, near, far=None):
    """
    Transforms camera-space rays to NDC space.
    
    Args:
        o: Tensor of shape (..., 3), ray origins in camera space.
        d: Tensor of shape (..., 3), ray directions in camera space.
        focal: Scalar, focal length.
        W: Scalar, image width in pixels.
        H: Scalar, image height in pixels.
        near: Scalar, near plane distance.
        far: Scalar, far plane distance. If None, assumes infinity.
        
    Returns:
        o_ndc: Tensor of shape (..., 3), NDC origins.
        d_ndc: Tensor of shape (..., 3), NDC directions.
    """
    # 1. Shift origin to near plane
    # t_n = -(near + o_z) / d_z
    # o_shifted = o + t_n * d
    # Note: We perform the shift before projection.
    # The formulas in S1 assume o is already at the near plane intersection.
    
    # Calculate t_n
    t_n = -(near + o[..., 2]) / d[..., 2]
    # Expand t_n for broadcasting
    t_n = t_n.unsqueeze(-1)
    
    # Shift origin
    o_shifted = o + t_n * d
    
    # 2. Compute constants
    # a_x, a_y
    ax = -focal / (W / 2.0)
    ay = -focal / (H / 2.0)
    
    # a_z, b_z
    if far is None:
        az = 1.0
        bz = 2.0 * near
    else:
        az = (far + near) / (far - near)
        bz = (2.0 * far * near) / (far - near)
        
    # 3. Project Origin
    # o'_x = ax * (o_x / o_z)
    # o'_y = ay * (o_y / o_z)
    # o'_z = az + bz / o_z
    
    ox_oz = o_shifted[..., 0] / o_shifted[..., 2]
    oy_oz = o_shifted[..., 1] / o_shifted[..., 2]
    
    o_ndc_x = ax * ox_oz
    o_ndc_y = ay * oy_oz
    o_ndc_z = az + bz / o_shifted[..., 2]
    
    o_ndc = torch.stack([o_ndc_x, o_ndc_y, o_ndc_z], dim=-1)
    
    # 4. Project Direction
    # d'_x = ax * (d_x/d_z - o_x/o_z)
    # d'_y = ay * (d_y/d_z - o_y/o_z)
    # d'_z = -bz / o_z
    
    dx_dz = d[..., 0] / d[..., 2]
    dy_dz = d[..., 1] / d[..., 2]
    
    d_ndc_x = ax * (dx_dz - ox_oz)
    d_ndc_y = ay * (dy_dz - oy_oz)
    d_ndc_z = -bz / o_shifted[..., 2]
    
    d_ndc = torch.stack([d_ndc_x, d_ndc_y, d_ndc_z], dim=-1)
    
    return o_ndc, d_ndc
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $d_z = 0$, the ray is parallel to the near plane and cannot be shifted to it. This is a degenerate case for forward-facing scenes. [G] In practice, rays are generated from pixel centers, so $d_z$ is rarely exactly zero, but numerical stability should be considered.
*   **Unbounded Scenes:** NDC is only suitable for unbounded scenes. Using it for bounded scenes can lead to incorrect sampling density [S3, unofficial].
*   **Coordinate System:** The derivation assumes the camera looks in the $-Z$ direction. If the camera looks in $+Z$, signs must be adjusted [S1].

## Subtopics

### [General NDC Ray Conversion](2429-general-ndc-ray-conversion.md)
General NDC Ray Conversion maps camera-space rays into a normalized device coordinate (NDC) space to fix the near and far clipping planes at constant values, removing the need for scene-dependent depth bounds in non-spherical datasets. Unlike Blender-specific NDC conventions that assume fixed intrinsics and sign conventions, this method handles arbitrary camera parameters by explicitly shifting ray origins to the near plane and projecting them based on focal length and image dimensions. It is primarily used for forward-facing scenes where depth ranges are large, enabling uniform sampling in NDC space to correspond to inverse depth (disparity) sampling in the original world space.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of rays. The transformation involves basic arithmetic operations per ray.
*   **Memory Complexity:** $O(N)$ to store the transformed rays.
*   **Strengths:** Allows rendering of unbounded scenes within a fixed bounding volume $[-1, 1]^3$. Enables uniform sampling in disparity, which is crucial for forward-facing real-world captures where depth varies significantly [S1][S3, unofficial].
*   **Limitations:** Not suitable for bounded scenes or scenes with significant depth variation that is not forward-facing [S3, unofficial]. Requires accurate near plane estimation.
*   **When to Use:** Forward-facing real-world scenes (e.g., LLFF dataset) where the background extends to infinity [S6, unofficial].
*   **When Not to Use:** Synthetic bounded scenes (e.g., Blender datasets) where standard ray generation is sufficient [S6, unofficial].

## Background
The NDC Ray Transformation was introduced in the NeRF paper (Mildenhall et al., 2020) to handle real-world forward-facing captures where the scene extends to infinity [S1]. It adapts the normalized device coordinates used in triangle rasterization pipelines to neural radiance fields, mapping the perspective rays to parallel rays in the transformed volume and using disparity instead of metric depth [S1]. This allows the neural network to learn a bounded representation for unbounded scenes.

## Sources
[S1] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S2] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S3] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S4] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S5] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S6] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S7] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S8] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S9] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S10] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
