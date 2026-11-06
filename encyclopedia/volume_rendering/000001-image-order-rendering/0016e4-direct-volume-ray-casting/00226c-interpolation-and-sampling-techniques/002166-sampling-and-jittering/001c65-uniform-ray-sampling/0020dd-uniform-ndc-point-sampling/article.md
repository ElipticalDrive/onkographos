# Uniform NDC Point Sampling

Also called: NDC Ray Sampling, NDC Sampling

## Overview
Uniform NDC Point Sampling transforms camera rays into an average camera frame and uniformly samples points within the projected Normalized Device Coordinates (NDC) space [S2]. It solves the problem of inefficient sample distribution in large scenes by providing a linear sampling in disparity from the near plane to infinity in original space, which combines the benefits of uniform and logarithmic sampling approaches [S2]. It is distinct from other strategies because it is only applicable if all samples lie strictly in the front hemisphere of the average camera frame, a limitation not shared by siblings like Stratified Ray Sampling [S2].

## How it works
**Inputs:**
*   `rays_o` (N, 3) ray origins and `rays_d` (N, 3) ray directions [S1, unofficial].
*   `xyz_min` (3) and `xyz_max` (3) bounding box limits [S1, unofficial].
*   `N_samples`: The fixed number of samples per ray [S1, unofficial][S3, unofficial].

**Outputs:**
*   `rays_pts` (N, N_samples, 3) sampled 3D coordinates [S1, unofficial].
*   `mask_outbbox` (N, N_samples) boolean mask indicating points outside the bounding box [S1, unofficial].

**Data Structures:**
*   Points are stored as a 3D tensor of shape (N, N_samples, 3) [S1, unofficial].

**Steps:**
1.  **Transform to Average Frame:** Transform ray origins and directions into the average camera frame [S2].
2.  **Compute Normalized Distance:** For each ray index $i_{ray} \in \{0, \dots, N-1\}$ and step index $i_{step} \in \{0, \dots, N_{samples}-1\}$, compute the normalized distance $dist = i_{step} / (N_{samples} - 1)$ [S1, unofficial][S3, unofficial].
3.  **Interpolate 3D Point:** Compute the 3D point coordinates by linear interpolation: $p_x = o_x + d_x \times dist$, $p_y = o_y + d_y \times dist$, $p_z = o_z + d_z \times dist$, where $(o, d)$ are the transformed ray origin and direction [S3, unofficial].
4.  **Bounding Box Check:** Determine if the point is outside the bounding box defined by $xyz_{min}$ and $xyz_{max}$ by checking if any coordinate is less than the min or greater than the max, populating `mask_outbbox` [S3, unofficial].

**Formulas:**
$$ dist = \frac{i_{step}}{N_{samples} - 1} $$
$$ p = o + d \times dist $$
where $i_{step}$ ranges from $0$ to $N_{samples}-1$, and $o, d$ are the transformed ray origin and direction vectors.

**Parameters:**
*   $N_{samples}$: The fixed number of samples per ray [S1, unofficial][S3, unofficial].
*   $d_{min} = 1$, $d_{max} = \infty$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_ndc_point_sampling(rays_o, rays_d, xyz_min, xyz_max, n_samples):
    """
    Samples points uniformly in NDC space along rays.
    
    Args:
        rays_o: List of tuples (ox, oy, oz) of length N
        rays_d: List of tuples (dx, dy, dz) of length N
        xyz_min: Tuple (xmin, ymin, zmin)
        xyz_max: Tuple (xmax, ymax, zmax)
        n_samples: Integer, number of samples per ray
        
    Returns:
        rays_pts: List of lists of tuples (px, py, pz) of shape (N, n_samples)
        mask_outbbox: List of lists of booleans of shape (N, n_samples)
    """
    # Step 1: Transform to average camera frame
    # [G] The exact transformation matrix is not provided in the sources.
    # Implementers must apply the average camera frame transformation defined in the original NeRF paper appendix.
    # For this implementation, we assume rays_o and rays_d are already transformed.
    
    # Step 2: Compute normalized distance
    dist = [i / (n_samples - 1) for i in range(n_samples)]
    
    # Step 3: Interpolate 3D points
    rays_pts = []
    for i in range(len(rays_o)):
        o = rays_o[i]
        d = rays_d[i]
        pts = []
        for j in range(n_samples):
            t = dist[j]
            pts.append((o[0] + d[0] * t, o[1] + d[1] * t, o[2] + d[2] * t))
        rays_pts.append(pts)
        
    # Step 4: Bounding box check
    mask_outbbox = []
    for i in range(len(rays_pts)):
        row_mask = []
        for j in range(n_samples):
            p = rays_pts[i][j]
            out = (p[0] < xyz_min[0] or p[0] > xyz_max[0] or
                   p[1] < xyz_min[1] or p[1] > xyz_max[1] or
                   p[2] < xyz_min[2] or p[2] > xyz_max[2])
            row_mask.append(out)
        mask_outbbox.append(row_mask)
        
    return rays_pts, mask_outbbox
```

**Edge Cases and Pitfalls:**
*   **Front Hemisphere Constraint:** The method is only applicable if all samples lie strictly in the front hemisphere of the average camera frame [S2].
*   **Perspective Distortions:** Large deviations from the average camera frame lead to high non-linear perspective distortions, which may negatively impact the learning process [S2].
*   **Sample Count Threshold:** In ablation studies, NDC sampling only surpasses uniform sampling in quality at 128 samples [S2]. It performs worse than all other local sampling approaches until $N=32$ [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \times N_{samples})$ per batch, as it performs a fixed number of linear interpolations [S1, unofficial].
*   **Memory Complexity:** $O(N \times N_{samples})$ to store the sampled points and bounding box masks [S1, unofficial].
*   **Performance:** In ablation studies on diverse scenes (Bulldozer, Forest, Classroom, San Miguel), NDC sampling only manages to surpass uniform sampling at 128 samples [S2]. It performs worse than all other local sampling approaches until $N=32$ [S2].
*   **Strengths:** Combines uniform and logarithmic sampling approaches from the perspective of network input [S2]. Linear in NDC space [S2].
*   **Limitations:** Limited to scenes where rays can be transformed to an average frame with all samples in the front hemisphere [S2]. Sensitive to large deviations from the average camera frame [S2].
*   **When to Use:** When using a fixed number of samples per ray and the scene geometry allows for a stable average camera frame transformation [S2].
*   **When Not to Use:** When rays extend into the back hemisphere of the average camera frame or when high accuracy is required with low sample counts (<128) [S2].

## Background
Uniform NDC Point Sampling was introduced in the context of NeRF (Neural Radiance Fields) [S2]. It is based on the NDC sampling strategy proposed in the original NeRF paper [MST*20] [S2]. The method transforms rays into an average camera frame and uniformly samples within the projected normalized device coordinates (NDC), directly feeding NDC samples into positional encoding [S2]. The detailed derivations for the transformation are provided in the appendix of the original NeRF paper [S2].

## Sources
[S1] sunset1995/DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
[S2] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
[S3] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
