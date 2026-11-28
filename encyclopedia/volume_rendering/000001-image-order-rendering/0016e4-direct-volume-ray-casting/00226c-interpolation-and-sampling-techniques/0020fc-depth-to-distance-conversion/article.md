# Depth to Distance Conversion

Also called: Depth to Distance, depth2dist, z_vals to dists

## Overview
Depth to Distance Conversion transforms sampled depth values ($z\_vals$) along a camera ray into the physical segment lengths ($dists$) between consecutive samples [S4, unofficial]. This conversion is necessary because uniform sampling in depth space does not correspond to uniform physical spacing in 3D space under perspective projection [S1, unofficial][S3, unofficial]. It accounts for the geometric relationship between the ray direction and the viewing angle [S1, unofficial][S3, unofficial]. Unlike [NDC Points to Distance](2500-ndc-points-to-distance.md), which operates in Normalized Device Coordinates, this method operates directly on camera-space depth values [S1, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `z_vals`, a tensor of shape $(N_{rays}, N_{samples})$ containing the depth values for each sample point along each ray [S2, unofficial].
*   **Output:** `dists`, a tensor of shape $(N_{rays}, N_{samples})$ containing the physical distance between consecutive samples [S2, unofficial].

**Data Structures:**
The input `z_vals` is a contiguous tensor where `z_vals[i, j]` represents the depth of the $j$-th sample along the $i$-th ray [S2, unofficial]. The values are assumed to be sorted in increasing order of depth (near to far) [G].

**Steps:**
1.  **Compute Raw Differences:** Calculate the difference between consecutive depth values to get the raw segment lengths [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial]. For a ray with samples $z_0, z_1, \dots, z_{N-1}$, the raw distance between sample $j$ and $j+1$ is $z_{j+1} - z_j$ [S1, unofficial].
2.  **Handle Boundary:** Append a large constant value ($10^{10}$) to the end of the distance array to represent the segment after the last sample [S2, unofficial][S3, unofficial][S4, unofficial]. This ensures the output tensor has the same shape as the input `z_vals` [S2, unofficial].
3.  **Cosine Scaling:** Scale the raw distances by the cosine of the angle between the ray direction and the viewing direction [S1, unofficial][S3, unofficial]. This step corrects for the perspective projection effect where depth intervals do not linearly map to Euclidean distances along the ray [S1, unofficial][S3, unofficial].

**Formulas:**
Let $z_{i,j}$ be the depth of the $j$-th sample on the $i$-th ray [G].
The raw distance $d'_{i,j}$ between sample $j$ and $j+1$ is:
$$ d'_{i,j} = z_{i,j+1} - z_{i,j} $$
For the last sample $j = N_{samples}-1$, the distance is set to a large constant $C = 10^{10}$ [S2, unofficial][S3, unofficial][S4, unofficial].
The final distance $d_{i,j}$ is scaled by $\cos \theta_i$, where $\theta_i$ is the angle between the ray direction $\mathbf{d}_i$ and the viewing direction (or the axis along which depth is measured) [S1, unofficial][S3, unofficial]:
$$ d_{i,j} = d'_{i,j} \cdot \cos \theta_i $$
If the rays are parallel to the viewing axis (orthographic), $\cos \theta = 1$ [G]. For perspective rays, this factor varies [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def depth_to_distance(z_vals, cos_angle):
    """
    Convert depth values (z_vals) to distances between samples.
    
    Args:
        z_vals: Tensor of shape (N_rays, N_samples) containing depth values.
        cos_angle: Tensor of shape (N_rays,) or (N_rays, 1) containing the cosine of the angle 
                   between the ray and the viewing direction.
    
    Returns:
        dists: Tensor of shape (N_rays, N_samples) containing segment distances.
    """
    # Step 1: Compute raw differences between consecutive samples
    dists_raw = z_vals[..., 1:] - z_vals[..., :-1]
    
    # Step 2: Append a large constant to the last segment to match input shape
    large_constant = torch.tensor(1e10, device=z_vals.device, dtype=z_vals.dtype)
    last_segment = large_constant * torch.ones_like(dists_raw[..., :1])
    dists = torch.cat([dists_raw, last_segment], dim=-1)
    
    # Step 3: Scale by cosine of the angle
    if cos_angle.dim() == 1:
        cos_angle = cos_angle.unsqueeze(-1)
    dists = dists * cos_angle
    
    return dists
```

**Parameters:**
*   `large_constant`: $10^{10}$ [S2, unofficial][S3, unofficial][S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Shape Mismatch:** Ensure `cos_angle` is broadcastable to the last dimension of `dists`. If `cos_angle` is per-ray, it must be unsqueezed to shape $(N_{rays}, 1)$ [G].
*   **Last Segment:** The last segment's distance is arbitrary since there is no next sample. The large constant is a convention to maintain tensor shape [S2, unofficial][S3, unofficial][S4, unofficial].
*   **Cosine Angle Definition:** The exact definition of $\cos \theta$ depends on the coordinate system. In many implementations, if the ray direction is normalized and the depth is measured along the camera's z-axis, this factor accounts for the projection geometry [G].

## Subtopics

### [NDC Points to Distance](2500-ndc-points-to-distance.md)
Converts sampled Normalized Device Coordinates (NDC) points to segment distances between consecutive samples along a ray, accounting for the cosine of the viewing angle. It operates directly in NDC space, where uniform sampling steps do not correspond to uniform physical distances, unlike linear depth-to-distance methods.

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \cdot N_{samples})$. The operation involves simple subtraction and multiplication, which are linear in the number of samples [G].
*   **Memory Complexity:** $O(N_{rays} \cdot N_{samples})$ for the output tensor `dists` [G].
*   **Strengths:** Simple, efficient, and numerically stable. It correctly accounts for perspective projection effects when combined with the cosine scaling [G].
*   **Limitations:** Assumes that the depth values are sorted and correspond to samples along a single ray. It does not handle non-linear depth mappings or complex coordinate transformations beyond the cosine scaling [G].
*   **When to Use:** When using depth-based sampling in volume rendering or neural radiance fields where samples are generated in camera space [G].

## Background
Not given in the sources.

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S2] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
[S3] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S4] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
