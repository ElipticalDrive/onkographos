# Uniform NDC Point Sampling

Also called: NDC Ray Sampling, NDC Sampling

## Overview
Uniform NDC Point Sampling generates a fixed number of points uniformly distributed along each ray in Normalized Device Coordinates (NDC) space, mapping the ray parameter $t$ from $0$ to $1$ [S1, S3, unofficial]. Unlike standard uniform sampling which depends on scene bounding boxes or depth ranges, this method assumes a canonical camera frame, making it suitable for neural rendering pipelines that require consistent positional encoding inputs regardless of scene scale [S2]. It sets itself apart from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by decoupling sample density from physical depth, and from [Non-linear (Logarithmic) Sampling](5814-non-linear-logarithmic-sampling.md) by providing a linear sampling curve in NDC space that corresponds to a $1/x$ disparity distribution in original space [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `rays_o`: Tensor of shape $(N, 3)$, ray origins in the average camera frame [S1, S3, unofficial].
    *   `rays_d`: Tensor of shape $(N, 3)$, ray directions in the average camera frame [S1, S3, unofficial].
    *   `N_samples`: Integer, the fixed number of samples per ray [S1, S3, unofficial].
    *   `xyz_min`, `xyz_max`: Tensors of shape $(3)$, defining the bounding box for validity checks (optional but common in implementations like DirectVoxGO) [S1, S3, unofficial].
    *   `ndc_near_plane`: Float, default $1.0$, defining the near clipping plane in NDC space [S4, unofficial].
*   **Outputs:**
    *   `rays_pts`: Tensor of shape $(N, N\_samples, 3)$, the 3D coordinates of the sampled points [S1, S3, unofficial].
    *   `mask_outbbox`: Tensor of shape $(N, N\_samples)$, boolean mask indicating points outside the bounding box [S1, S3, unofficial].

**Data Structures:**
The method operates on standard floating-point tensors [G]. The ray parameter $t$ (or $dist$) is a scalar value in $[0, 1]$ [S1, S3, unofficial].

**Steps:**
1.  **Transform to Average Camera Frame:** Before sampling, rays must be transformed into an "average" camera frame [S2]. This involves rotating and translating the ray origins and directions such that the camera poses are centered and aligned [S2]. This step is critical; if the original rays are not in this frame, the NDC assumption fails [S2].
2.  **Compute Normalized Distance:** For each ray $i$ and sample index $j$ (where $j \in \{0, \dots, N\_samples-1\}$), compute the normalized distance $dist$:
    $$ dist = \frac{j}{N\_samples - 1} $$
    This maps the samples linearly from $t=0$ to $t=1$ [S1, S3, unofficial].
3.  **Compute 3D Coordinates:** Calculate the 3D point $p$ for each sample using linear interpolation along the ray:
    $$ p_x = o_x + d_x \times dist $$
    $$ p_y = o_y + d_y \times dist $$
    $$ p_z = o_z + d_z \times dist $$
    where $(o_x, o_y, o_z)$ and $(d_x, d_y, d_z)$ are the components of the ray origin and direction in the average camera frame [S1, S3, unofficial].
4.  **Bounding Box Check (Optional):** If `xyz_min` and `xyz_max` are provided, check if any coordinate of $p$ is less than the corresponding min or greater than the corresponding max [S3, unofficial]. Set the corresponding bit in `mask_outbbox` to true if the point is outside [S3, unofficial].
5.  **Distance Scaling (Variant):** In implementations like TensoRF, if NDC sampling is enabled, inter-sample distances are computed and scaled by the ray direction norm to approximate Euclidean distance [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def uniform_ndc_point_sampling(rays_o, rays_d, N_samples, xyz_min=None, xyz_max=None):
    """
    Samples N_samples uniformly spaced points along each ray in NDC space.
    
    Args:
        rays_o: Tensor of shape (N, 3), ray origins in average camera frame.
        rays_d: Tensor of shape (N, 3), ray directions in average camera frame.
        N_samples: Integer, number of samples per ray.
        xyz_min: Tensor of shape (3), minimum bounding box coordinates.
        xyz_max: Tensor of shape (3), maximum bounding box coordinates.
        
    Returns:
        rays_pts: Tensor of shape (N, N_samples, 3), sampled points.
        mask_outbbox: Tensor of shape (N, N_samples), True if point is outside bbox.
    """
    N = rays_o.shape[0]
    
    # Step 2: Compute normalized distance
    # dist ranges from 0.0 to 1.0, matching j / (N_samples - 1)
    dists = torch.linspace(0.0, 1.0, N_samples, device=rays_o.device)
    
    # Expand dimensions for broadcasting
    # rays_o: (N, 3) -> (N, 1, 3)
    # rays_d: (N, 3) -> (N, 1, 3)
    # dists: (N_samples,) -> (1, N_samples, 1)
    rays_o_exp = rays_o.unsqueeze(1)
    rays_d_exp = rays_d.unsqueeze(1)
    dists_exp = dists.unsqueeze(0).unsqueeze(2)
    
    # Step 3: Compute 3D coordinates
    # p = o + d * dist
    rays_pts = rays_o_exp + rays_d_exp * dists_exp
    
    mask_outbbox = torch.zeros((N, N_samples), dtype=torch.bool, device=rays_o.device)
    
    # Step 4: Bounding box check
    if xyz_min is not None and xyz_max is not None:
        # Check if any coordinate is less than min or greater than max
        # xyz_min: (3,) -> (1, 1, 3)
        # xyz_max: (3,) -> (1, 1, 3)
        xyz_min_exp = xyz_min.unsqueeze(0).unsqueeze(0)
        xyz_max_exp = xyz_max.unsqueeze(0).unsqueeze(0)
        
        out_min = rays_pts < xyz_min_exp
        out_max = rays_pts > xyz_max_exp
        
        # Any dimension out of bounds
        out_bbox = out_min | out_max
        
        # Reduce over the last dimension (x, y, z)
        mask_outbbox = out_bbox.any(dim=-1)
        
    return rays_pts, mask_outbbox
```

**Parameters:**
*   `N_samples`: The fixed number of samples per ray [S1, S3, unofficial].
*   `ndc_near_plane`: Default $1.0$. Defines the near clipping plane in NDC space. Used in distance scaling variants [S4, unofficial].
*   `d_min`: $1$. Lower bound for NDC depth [S2].
*   `d_max`: $\infty$. Upper bound for NDC depth [S2].

**Edge Cases and Pitfalls:**
*   **Average Camera Frame:** The method is only applicable if all samples lie strictly in the front hemisphere of the average camera frame [S2]. Large deviations from this frame lead to high non-linear perspective distortions, which can negatively impact the learning process [S2].
*   **Ray Direction Norm:** In NDC space, the ray direction norm may vary [S4, unofficial]. If Euclidean distance accuracy is required, distances must be scaled by the ray direction norm [S4, unofficial].
*   **Bounding Box:** If the bounding box is not provided, the method does not filter out-of-bounds points, which may lead to sampling empty space or invalid regions [S3, unofficial].

## Subtopics

### NDC Ray Sampling & Distance Scaling
A variant that computes inter-sample distances and scales them by the ray direction norm to approximate Euclidean distance in NDC space [S4, unofficial]. It uses parameters `ndc_near_plane` (default 1.0), `simulate_euclid_sample` (bool), and `simulate_euclid_depth` (float) to control depth simulation [S4, unofficial]. This variant is chosen when accurate depth-based weighting is needed despite using NDC sampling [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \times N\_samples)$, where $N$ is the number of rays and $N\_samples$ is the number of samples per ray [S1, S3, unofficial].
*   **Memory Complexity:** $O(N \times N\_samples \times 3)$ for storing the sampled points [S1, unofficial].
*   **Performance:** NDC sampling only surpasses uniform sampling in quality at 128 samples in ablation studies [S2].
*   **Strengths:** Consistent sample distribution regardless of scene scale; simple linear interpolation; compatible with positional encoding that expects NDC inputs [S2].
*   **Limitations:** Requires transformation to an average camera frame; sensitive to large deviations from the average frame; less effective than logarithmic or local sampling for foreground-focused scenes with low sample counts [S2].
*   **When to Use:** When using positional encodings designed for NDC space; when a fixed number of samples is required regardless of depth [S2].
*   **When Not to Use:** When the scene cannot be easily transformed to an average camera frame; when samples must remain strictly in the front hemisphere of the average camera frame; when depth-aware sampling (e.g., logarithmic) is preferred [S2].

## Background
Uniform NDC Point Sampling was introduced in the context of NeRF (Neural Radiance Fields) to address positional encoding applied equally to foreground and background frequencies [S2]. The method transforms rays into an average camera frame and samples uniformly in NDC space, allowing the network to learn consistent features regardless of the original camera configuration [S2]. It was proposed as an alternative to logarithmic sampling, offering a linear sampling curve in NDC space that corresponds to a $1/x$ disparity distribution in original space [S2]. The approach is detailed in the original NeRF paper and subsequent works like DONeRF [S2].

## Sources
[S1] DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
[S2] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
[S3] ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S4] Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
