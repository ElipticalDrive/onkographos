# Projection Jacobian Computation

Also called: perspective projection Jacobian

## Overview
Computes the $2 \times 3$ Jacobian matrix of the perspective projection function with respect to 3D coordinates, used for linearizing the projection of 3D Gaussians into screen space [S1, unofficial]. It transforms the 3D covariance structure into 2D screen-space covariance, enabling accurate Gaussian splatting [S1, unofficial].

## How it works
**Inputs:**
- `means_3D`: Tensor of shape $(N, 3)$ containing 3D Gaussian centers in world space [S1, unofficial].
- `camera`: Camera object providing view transformation and focal lengths [S1, unofficial].
- `img_size`: Tuple $(W, H)$ representing image width and height [S1, unofficial].

**Outputs:**
- `J`: Tensor of shape $(N, 2, 3)$ containing the Jacobian matrix for each Gaussian [S1, unofficial].

**Data Structures:**
- `means_view_space`: Tensor of shape $(N, 3)$, view-space coordinates $(t_x, t_y, t_z)$ [S1, unofficial].
- `J`: Tensor of shape $(N, 2, 3)$, initialized to zeros [S1, unofficial].

**Steps:**
1.  **Validate Camera:** Check if the camera is in NDC mode. If `camera.in_ndc()` is true, raise a `RuntimeError` [S1, unofficial].
2.  **Extract Parameters:** Get focal lengths $f_x, f_y = \text{camera.focal\_length.flatten()}$ and image dimensions $W, H = \text{img\_size}$ [S1, unofficial].
3.  **Transform to View Space:** Apply view transformation to 3D means: $\text{means\_view\_space} = \text{view\_transform.transform\_points}(\text{means\_3D})$ [S1, unofficial]. Extract components: $t_x = \text{means\_view\_space}[:, 0]$, $t_y = \text{means\_view\_space}[:, 1]$, $t_z = \text{means\_view\_space}[:, 2]$ [S1, unofficial]. Compute squared depth: $t_z^2 = t_z \cdot t_z$ [S1, unofficial].
4.  **Compute FOV Limits:** Calculate half-tangent of field of view: $\text{half\_tan\_fov\_x} = 0.5 \cdot W / f_x$ and $\text{half\_tan\_fov\_y} = 0.5 \cdot H / f_y$ [S1, unofficial]. Apply safety margin (1.3) to define clamping limits: $\text{lim\_x} = 1.3 \cdot \text{half\_tan\_fov\_x}$ and $\text{lim\_y} = 1.3 \cdot \text{half\_tan\_fov\_y}$ [S1, unofficial].
5.  **Clamp Normalized Coordinates:** To prevent division by zero or extreme values near frustum edges, clamp the normalized coordinates $(t_x/t_z, t_y/t_z)$ before scaling back: $t_x^{\text{clamp}} = \text{clamp}(t_x / t_z, -\text{lim\_x}, \text{lim\_x}) \cdot t_z$ and $t_y^{\text{clamp}} = \text{clamp}(t_y / t_z, -\text{lim\_y}, \text{lim\_y}) \cdot t_z$ [S1, unofficial].
6.  **Initialize Jacobian:** Create tensor `J` of shape $(N, 2, 3)$ filled with zeros [S1, unofficial].
7.  **Fill Jacobian Entries:** Assign values using the clamped coordinates: $J[:, 0, 0] = f_x / t_z$, $J[:, 1, 1] = f_y / t_z$, $J[:, 0, 2] = -(f_x \cdot t_x^{\text{clamp}}) / t_z^2$, and $J[:, 1, 2] = -(f_y \cdot t_y^{\text{clamp}}) / t_z^2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_projection_jacobian(means_3D, camera, view_transform, img_size):
    """
    Computes the 2x3 Jacobian matrix for perspective projection of 3D Gaussians.
    
    Args:
        means_3D: Tensor of shape (N, 3), 3D Gaussian centers.
        camera: Camera object with methods focal_length, in_ndc.
        view_transform: Object with method transform_points.
        img_size: Tuple (W, H), image dimensions.
        
    Returns:
        J: Tensor of shape (N, 2, 3), the Jacobian matrix.
    """
    # Step 1: Validate Camera
    if camera.in_ndc():
        raise RuntimeError("Camera must not be in NDC mode for this Jacobian computation.")
    
    # Step 2: Extract Parameters
    focal_lengths = camera.focal_length.flatten()
    fx, fy = focal_lengths[0], focal_lengths[1]
    W, H = img_size
    
    # Step 3: Transform to View Space
    means_view_space = view_transform.transform_points(means_3D)
    
    tx = means_view_space[:, 0]
    ty = means_view_space[:, 1]
    tz = means_view_space[:, 2]
    tz2 = tz * tz
    
    # Step 4: Compute FOV Limits
    half_tan_fov_x = 0.5 * W / fx
    half_tan_fov_y = 0.5 * H / fy
    
    safety_margin = 1.3
    lim_x = safety_margin * half_tan_fov_x
    lim_y = safety_margin * half_tan_fov_y
    
    # Step 5: Clamp Normalized Coordinates
    norm_tx = tx / tz
    norm_ty = ty / tz
    
    norm_tx_clamped = torch.clamp(norm_tx, -lim_x, lim_x)
    norm_ty_clamped = torch.clamp(norm_ty, -lim_y, lim_y)
    
    tx_clamped = norm_tx_clamped * tz
    ty_clamped = norm_ty_clamped * tz
    
    # Step 6: Initialize Jacobian
    N = means_3D.shape[0]
    J = torch.zeros((N, 2, 3), dtype=means_3D.dtype, device=means_3D.device)
    
    # Step 7: Fill Jacobian Entries
    J[:, 0, 0] = fx / tz
    J[:, 1, 1] = fy / tz
    J[:, 0, 2] = -(fx * tx_clamped) / tz2
    J[:, 1, 2] = -(fy * ty_clamped) / tz2
    
    return J
```

**Formula:**
$$ J = \begin{bmatrix} f_x/t_z & 0 & -f_x t_x^{\text{clamp}}/t_z^2 \\ 0 & f_y/t_z & -f_y t_y^{\text{clamp}}/t_z^2 \end{bmatrix} $$
where $f_x, f_y$ are focal lengths, $t_x^{\text{clamp}}, t_y^{\text{clamp}}$ are the clamped view-space coordinates from Step 5, $t_z$ is depth, and $t_z^2 = t_z \cdot t_z$ [S1, unofficial].

**Parameters:**
- **FOV Safety Margin:** 1.3 multiplier applied to half-tan-FOV limits to clamp normalized coordinates and stabilize Jacobian computation near frustum edges [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** Division by $t_z$ can cause instability if points are at or behind the camera; clamping normalized coordinates mitigates extreme values [S1, unofficial].
- **Frustum Edges:** Without clamping, normalized coordinates near the frustum boundary can lead to large Jacobian values, causing numerical instability in covariance transformation [S1, unofficial].

## Subtopics

### Per-Pixel Jacobian Evaluation
A variant where the Jacobian is evaluated at each ray–splat intersection point rather than at the surfel center, yielding a per-pixel MIP kernel that is maximally accurate but computationally expensive [S2, unofficial]. It computes the full $2 \times 2$ Jacobian at every covered pixel, followed by the outer product and matrix inverse [S2, unofficial]. This method incurs a rendering time of 65 ms per frame, which is a ~1.3x overhead relative to the center-precomputed method and a ~55% increase over the unfiltered baseline [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The per-pixel variant renders in 65 ms per frame, which is a ~1.3x overhead relative to the center-precomputed method and a ~55% increase over the unfiltered baseline [S2, unofficial].
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
