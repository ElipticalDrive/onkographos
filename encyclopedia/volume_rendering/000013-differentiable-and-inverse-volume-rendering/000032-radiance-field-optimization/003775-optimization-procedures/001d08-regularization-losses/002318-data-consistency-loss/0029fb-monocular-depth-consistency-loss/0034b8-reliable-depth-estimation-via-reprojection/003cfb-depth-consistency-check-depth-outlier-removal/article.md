# Depth Consistency Check / Depth Outlier Removal

Also called: Geometric Consistency Masking, Reprojection-based Depth Filtering

## Overview
This method filters noisy depth data by verifying the geometric consistency of unprojected depth maps across multiple views. It solves the problem of outliers (such as floaters or incorrect reconstructions) in monocular depth estimation by ensuring that a depth value, when transformed to a neighboring view, remains consistent with the observed depth in that neighbor. It differs from sibling methods like [Depth Supervision Loss (Laug)](8111-depth-supervision-loss-laug.md) by operating as a pre-processing or auxiliary consistency check rather than a direct differentiable loss term, and from [Reliable Depth Estimation via Reprojection](7293-reliable-depth-estimation-via-reprojection.md) by focusing on hard filtering via a fixed threshold rather than probabilistic weighting.

## How it works
**Inputs:**
- Depth maps $D_i$ and $D_j$ for two views $i$ and $j$.
- Camera intrinsics and extrinsics for views $i$ and $j$.
- Threshold $\sigma$.

**Outputs:**
- A geometric consistency mask $M_d$ identifying valid depth pixels.

**Data Structures:**
- $D_i$: 2D array of depth values for view $i$.
- $M_d$: 2D boolean or binary mask of the same spatial dimensions as $D_i$.

**Steps:**
1. **Unproject:** Convert the depth map $D_i$ of frame $i$ into a 3D point cloud. For each pixel $\mathbf{u}_i$ with depth $D_i(\mathbf{u}_i)$, compute the 3D position $\mathbf{P}_i = \pi_i^{-1}(\mathbf{u}_i, D_i(\mathbf{u}_i))$, where $\pi_i^{-1}$ is the inverse projection function using the camera intrinsics of view $i$.
2. **Reproject:** Transform the 3D points $\mathbf{P}_i$ into the coordinate system of a nearby view $j$. Project these points onto view $j$'s image plane to obtain the projected pixel coordinates $\mathbf{u}_{i \to j} = \pi_j(\mathbf{P}_i)$.
3. **Sample:** Retrieve the depth value $D_j$ at the corresponding pixel location in view $j$. The projected depth is defined as $D_{i \to j} = D_j(\mathbf{u}_{i \to j})$. Note: If $\mathbf{u}_{i \to j}$ falls outside the image bounds or corresponds to an invalid depth in $D_j$, the pixel is considered inconsistent.
4. **Compare:** Calculate the absolute difference between the reprojected depth $D_{i \to j}$ and the observed depth $D_j$ at that location.
5. **Filter:** Apply the consistency threshold $\sigma$. If the error is within the threshold, the pixel is marked as consistent.

**Formula:**
The geometric consistency mask $M_d$ is formulated as [S1]:
$$
M_d = \left| D_{i \to j} - D_j \right| < \sigma
$$
where:
- $D_{i \to j} = D_j(\pi_j \pi_i^{-1}(\mathbf{u}_i))$ is the reprojected depth.
- $\mathbf{u}_i$ denotes the pixel coordinates of the $i$-th frame.
- $M_d$ represents the geometric consistency mask (true if consistent, false otherwise).
- $\sigma$ is the empirical threshold for the absolute error.

**Parameters:**
- $\sigma = 0.2$ m: Empirical threshold for the absolute relative error in depth reprojection [S1].

**Edge Cases and Pitfalls:**
- **Occlusions:** If view $j$ occludes the surface visible in view $i$, $D_j$ will be closer to the camera than $D_{i \to j}$. The method may incorrectly flag these as outliers unless occlusion handling is explicitly added. The sources do not specify occlusion handling logic beyond the basic threshold check [S1].
- **Invalid Pixels:** Pixels in $D_j$ that are invalid (e.g., zero or infinity) must be handled. The sources do not specify the exact behavior for invalid $D_j$ values, but logically they should be excluded from the mask [G].
- **Precision:** Floating-point errors in reprojection can cause small discrepancies. The threshold $\sigma$ must be large enough to accommodate these errors but small enough to filter true outliers.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_consistency_check(D_i, D_j, cam_i, cam_j, sigma=0.2):
    """
    Filters noisy depth data by unprojecting D_i to 3D and reprojecting to view j.
    
    Args:
        D_i: 2D array, depth map of frame i.
        D_j: 2D array, depth map of frame j.
        cam_i: Camera object for frame i (intrinsics, extrinsics).
        cam_j: Camera object for frame j (intrinsics, extrinsics).
        sigma: Float, threshold for absolute error (default 0.2m).
        
    Returns:
        M_d: 2D boolean array, geometric consistency mask.
    """
    # 1. Unproject D_i to 3D
    # Get pixel coordinates (u, v) for all valid pixels in D_i
    valid_pixels_i = get_valid_pixel_coords(D_i)
    
    # Convert depth to 3D points
    # P_i = cam_i.inverse_project(u, v, D_i[u,v])
    points_3d = []
    for u, v in valid_pixels_i:
        depth_val = D_i[u, v]
        if depth_val <= 0:
            continue
        p_3d = cam_i.inverse_project(u, v, depth_val)
        points_3d.append((u, v, p_3d))
        
    # 2. Reproject to view j
    M_d = np.zeros_like(D_i, dtype=bool)
    
    for u, v, p_3d in points_3d:
        # Project 3D point to view j
        u_proj, v_proj, depth_proj = cam_j.project(p_3d)
        
        # Check if projected pixel is within bounds of D_j
        if not (0 <= u_proj < D_j.shape[1] and 0 <= v_proj < D_j.shape[0]):
            continue
            
        # Get observed depth in view j
        d_j_obs = D_j[int(v_proj), int(u_proj)]
        
        # Check if observed depth is valid
        if d_j_obs <= 0:
            continue
            
        # 3. Compare depths
        error = abs(depth_proj - d_j_obs)
        
        # 4. Filter
        if error < sigma:
            M_d[u, v] = True
            
    return M_d
```

## Subtopics

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of valid pixels in $D_i$ and the number of neighbor views checked. Each pixel requires one unprojection and one reprojection.
- **Memory Complexity:** Requires storing the 3D point cloud for the current frame during the check, which is $O(N)$ where $N$ is the number of pixels.
- **Strengths:** Effective at removing gross outliers and floaters that do not satisfy geometric consistency across views. Simple to implement.
- **Limitations:** Relies on accurate depth maps in both views. Fails in occluded regions or textureless areas where depth estimation is unreliable. The fixed threshold $\sigma$ may not adapt to varying depth scales or noise levels across the scene.
- **When to use:** As a pre-processing step or auxiliary check in multi-view reconstruction pipelines where geometric consistency is critical.
- **When not to use:** When occlusions are frequent and not handled, or when depth maps are extremely noisy such that the threshold $\sigma$ cannot distinguish noise from valid geometry.

## Background
The method originates from [EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis](https://doi.org/10.1109/CVPR52734.2025.01054) [S1], published in 2025. It is used in the context of urban view synthesis to clean up depth data before or during the optimization of Gaussian splats, specifically to remove clear floaters in unprojected point clouds. The authors note that they further utilize a 3D statistical filter in Open3D to remove remaining floaters [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
