# Depth Consistency Check / Depth Outlier Removal

Also called: Geometric Consistency Masking, Reprojection-based Depth Filtering

## Overview
This method filters noisy depth data by unprojecting depth maps to 3D space and reprojecting them into nearby views to verify geometric consistency [S1]. It solves the problem of incorrect depth supervision caused by moving objects, ill-posed views, and occlusions in street view scenarios [S1]. It sets itself apart from [Depth Supervision Loss (Laug)](8111-depth-supervision-loss-laug.md) by using explicit geometric reprojection checks rather than direct loss minimization, and from [Multi-view Consistency Depth Filtering](12319-multi-view-consistency-depth-filtering.md) by focusing on pairwise reprojection error thresholds rather than complex multi-view voting or probabilistic models [S1, S2].

## How it works
**Inputs:**
- Depth map $D_i$ for source frame $i$.
- Camera extrinsics and intrinsics for source frame $i$ and target frame $j$.
- Pixel coordinates $\mathbf{u}_i$ for frame $i$.
- Thresholds: $\sigma$ (absolute error), $\delta$ (image space), $\varepsilon$ (relative depth), $N_{geo}$ (min views).

**Outputs:**
- Geometric consistency mask $M_d$ (binary or boolean array same size as $D_i$).
- Filtered depth values (where $M_d$ is true).

**Data Structures:**
- $D_i$: 2D array of depth values.
- $M_d$: 2D boolean array.
- $\mathbf{u}_i$: 2D array of pixel coordinates $(u, v)$.

**Steps:**
1. **Unproject:** Convert the depth map $D_i$ of frame $i$ into 3D points using the inverse projection matrix $\pi_i^{-1}$. For each pixel $\mathbf{u}_i$, compute the 3D point $\mathbf{X}_i = \pi_i^{-1}(\mathbf{u}_i, D_i(\mathbf{u}_i))$ [S1].
2. **Reproject:** Project the 3D points $\mathbf{X}_i$ into the coordinate system of a nearby view $j$ using the projection matrix $\pi_j$. This yields the projected pixel coordinates and the projected depth $D_{i \to j}$ [S1].
   - Formula: $D_{i \to j} = D_j(\pi_j \pi_i^{-1}(\mathbf{u}_i))$ [S1]. Note: $D_j$ is sampled at the projected pixel location.
3. **Compare:** Calculate the absolute difference between the reprojected depth $D_{i \to j}$ and the actual depth $D_j$ at the corresponding pixel in view $j$ [S1].
4. **Filter:** Apply the consistency check. A pixel is considered consistent if the error is below the threshold [S1].

**Formulas:**
The primary consistency mask $M_d$ is defined as [S1]:
$$ M_d = |D_{i \to j} - D_j| < \sigma $$
where:
- $D_{i \to j}$ is the reprojected depth from frame $i$ to frame $j$.
- $D_j$ is the ground truth or estimated depth in frame $j$ at the projected location.
- $\sigma$ is the empirical threshold for absolute relative error.

**Parameters:**
- $\sigma = 0.2$ m: Empirical threshold for the absolute error in depth reprojection [S1].
- $\delta = 1$: Image space consistency threshold (pixel distance) [S2].
- $\varepsilon = 0.01$: Depth consistency threshold (relative error) [S2].
- $N_{geo}$: Minimum number of source views required for consistency acceptance [S2].

**Edge Cases and Pitfalls:**
- **Occlusions:** If a point is occluded in view $j$ but visible in view $i$, the reprojection will land on a different surface or background, causing a large error. This method filters such outliers [S1].
- **Moving Objects:** Dynamic objects cause inconsistencies between frames. The reprojection check identifies these as outliers [S1].
- **Interpolation:** When retrieving depth $d_i$ or $d_{reproj}$, interpolation is required if projected coordinates are non-integer. This introduces small errors [S2].
- **Precision:** Floating-point precision in matrix multiplications for reprojection can accumulate errors. Using a 3D statistical filter (e.g., Open3D) on unprojected point clouds can remove clear floaters before reprojection [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_consistency_check(D_i, D_j, K_i, K_j, sigma=0.2, delta=1.0, epsilon=0.01, N_geo=1):
    """
    Perform depth consistency check between frame i and frame j.
    Supports both simple absolute error check (S1) and multi-view geometric filtering (S2).
    """
    # 1. Unproject D_i to 3D
    # X_i is a 3D point cloud corresponding to pixels in D_i
    X_i = unproject_depth(D_i, K_i)
    
    # 2. Reproject X_i to frame j
    # Get projected pixel coordinates and depth in frame j
    u_proj, v_proj, D_reproj = project_points(X_i, K_j, K_i)
    
    # 3. Sample D_j at projected locations
    # Handle boundary checks and interpolation
    D_j_sampled = sample_depth(D_j, u_proj, v_proj)
    
    # 4. Calculate errors
    abs_error = abs(D_reproj - D_j_sampled)
    
    # 5. Apply threshold (S1 method)
    mask_simple = abs_error < sigma
    
    # 6. Apply geometric filtering (S2 method)
    # Calculate image space distance
    # Note: u_proj, v_proj are floats; need to compare with integer pixel p
    # For simplicity, assume p is the integer coordinate of the original pixel in i
    # But S2 describes projecting p from reference to source, then back.
    # Here we are doing i -> j. Let's adapt S2 logic for pairwise i->j.
    
    # S2 Logic:
    # p is pixel in reference (i). pi is pixel in source (j).
    # Here i is reference, j is source.
    # Project p (from i) to pi (in j).
    # Retrieve depth di in source (j) -> this is D_j_sampled.
    # Re-project pi back to reference (i) -> this is D_reproj.
    # Check |preproj - p| < delta and |dreproj - d0|/d0 < epsilon.
    
    # Calculate image space distance between original pixel and reprojected pixel
    # We need the original pixel coordinates in i
    pixel_coords_i = get_pixel_coordinates(D_i)
    image_dist = sqrt((u_proj - pixel_coords_i[:, 0])**2 + (v_proj - pixel_coords_i[:, 1])**2)
    
    # Calculate relative depth error
    relative_depth_error = abs_error / D_i
    
    # Apply S2 thresholds
    mask_geo = (image_dist < delta) & (relative_depth_error < epsilon)
    
    # Combine masks or choose one. S1 uses mask_simple. S2 uses mask_geo and N_geo.
    # For this function, we return both for flexibility.
    return mask_simple, mask_geo

def multi_view_geometric_filter(D_ref, D_sources, K_ref, K_sources, delta=1.0, epsilon=0.01, N_geo=1):
    """
    Accept estimations as reliable if consistent in at least N_geo source views (S2).
    """
    total_views = len(D_sources)
    consistency_counts = np.zeros_like(D_ref, dtype=int)
    
    for i in range(total_views):
        D_src = D_sources[i]
        K_src = K_sources[i]
        
        # Unproject reference depth
        X_ref = unproject_depth(D_ref, K_ref)
        
        # Reproject to source view
        u_proj, v_proj, D_reproj = project_points(X_ref, K_src, K_ref)
        
        # Sample source depth
        D_src_sampled = sample_depth(D_src, u_proj, v_proj)
        
        # Calculate errors
        abs_error = abs(D_reproj - D_src_sampled)
        relative_error = abs_error / D_ref
        
        # Image space distance
        pixel_coords_ref = get_pixel_coordinates(D_ref)
        image_dist = sqrt((u_proj - pixel_coords_ref[:, 0])**2 + (v_proj - pixel_coords_ref[:, 1])**2)
        
        # Check consistency
        is_consistent = (image_dist < delta) & (relative_error < epsilon)
        
        # Count consistent views
        consistency_counts += is_consistent.astype(int)
        
    # Final mask
    final_mask = consistency_counts >= N_geo
    
    return final_mask
```

## Subtopics

### [Probabilistic Depth Testing](11488-probabilistic-depth-testing.md)
Probabilistic Depth Testing determines which input view's reprojection contains the correct geometric information for a novel view pixel by calculating the probability that its depth is closer than all other views' depths. It differs from hard visibility tests by using a mixture model with triangular distributions to perform soft depth resolution, which is necessary because point splatting lacks mesh connectivity and soft rasterization introduces uncertainty from noisy Multi-View Stereo (MVS) data. See [Probabilistic Depth Testing](11488-probabilistic-depth-testing.md).

### [Reprojection Confidence Measurement](15150-reprojection-confidence-measurement.md)
Reprojection Confidence Measurement is a method to measure depth accuracy and locate outliers by warping pixels from source images to target images and comparing similarities. It addresses the problem of wrong depth supervisions caused by moving objects, ill-posed views, and occlusions in street view scenarios. It sets itself apart from siblings like [Probabilistic Depth Testing](11488-probabilistic-depth-testing.md) by using deterministic geometric warping and multi-level similarity metrics (RGB, SSIM, VGG, flow) rather than probabilistic distributions. See [Reprojection Confidence Measurement](15150-reprojection-confidence-measurement.md).

### Geometric Consistency Filtering
Geometric Consistency Filtering measures the consistency of depth estimates among multiple views [S2]. For each pixel $p$ in the reference view with depth $d_0$, it projects to a pixel $p_i$ in the $i$-th source view, retrieves depth $d_i$, and reprojects back to get $d_{reproj}$ at $p_{reproj}$ [S2]. It accepts the estimation as reliable if $|p_{reproj} - p| < \delta$ and $|d_{reproj} - d_0|/d_0 < \varepsilon$ [S2]. It requires consistency in at least $N_{geo}$ source views [S2]. Parameters: $\delta = 1$, $\varepsilon = 0.01$, $N_{geo}$ varies [S2].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of pixels and $V$ is the number of source views checked. Each pixel requires unprojection, reprojection, and sampling.
- **Memory Complexity:** $O(N)$ for storing depth maps and masks. Unprojected 3D points require $O(N)$ additional memory.
- **Measured Performance:** In [S1], the method is used to remove noisy depth data and clear floaters in unprojected point clouds. It uses a 3D statistical filter (Open3D) with 20 neighbors and a standard deviation ratio of 2.0 to further clean the data [S1].
- **Strengths:** Effective at removing outliers caused by occlusions, moving objects, and MVS noise [S1]. Simple to implement and understand.
- **Limitations:** Relies on accurate camera calibration. Fails in textureless regions where depth estimation is inherently noisy. The threshold $\sigma$ is empirical and may need tuning for different scenes [S1].
- **When to use:** When using monocular depth estimators or MVS outputs that contain significant noise or outliers, especially in dynamic scenes [S1].
- **When not to use:** When high precision is required in textureless regions or when computational resources are extremely limited and simple thresholding is insufficient.

## Background
The method originates from the need to clean depth data for volume-based Gaussian Splatting and multi-view reconstruction. [S1] introduces the basic reprojection consistency check for EVolSplat, an efficient volume-based Gaussian Splatting method for urban view synthesis. [S2] adapts and expands this concept for VolRecon, using geometric consistency filtering for generalizable multi-view reconstruction. The approach builds on traditional multi-view stereo techniques [S2].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] VolRecon: Volume Rendering of Signed Ray Distance Functions for Generalizable Multi-View Reconstruction, https://doi.org/10.1109/cvpr52729.2023.01601, fetched 2026-10-26
