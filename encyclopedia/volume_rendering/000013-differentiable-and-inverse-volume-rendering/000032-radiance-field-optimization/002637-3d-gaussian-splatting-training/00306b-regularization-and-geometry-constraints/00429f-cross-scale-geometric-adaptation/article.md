# Cross-scale geometric adaptation

Also called: Lgeo, Reprojection error supervision, Reprojection-based cross-scale geometric adaptation

## Overview
Cross-scale geometric adaptation is a self-adaptive regularization technique that uses reprojection errors across multiple voxel scales to guide geometry learning without requiring explicit depth ground truth [S1]. It selects render depths with minimal reprojection error across different scales as pseudo labels, ensuring geometric integrity and reducing floaters more effectively than single-scale depth priors [S1]. Unlike siblings that rely on fixed schedules or external priors, this method dynamically adapts to scene frequencies by leveraging low-resolution voxels for coarse geometry early in training and high-resolution voxels for detail refinement later [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training views (frames $i$ and $j$), camera intrinsics $K_i, K_j$, camera poses (rotation $R$, translation $t$) for frames $i$ and $j$, multi-scale voxel representations (low, mid, high resolution), and current Gaussian parameters $\theta$.
*   **Outputs:** Geometric adaptation loss $\mathcal{L}_{geo}$ and updated Gaussian parameters via backpropagation.

**Data Structures:**
*   **Multi-scale Voxels:** A hierarchy of voxel grids at $L$ scales (e.g., $l=0$ for low, $l=1$ for mid, $l=2$ for high). Each scale $l$ provides a depth map $D_l$ and color map $C_l$ via volume rendering.
*   **Pixel Coordinates:** 2D pixel coordinate $p_i$ in frame $i$ and its homogeneous augmentation $\tilde{p}_i = [u, v, 1]^T$.

**Numbered Steps:**
1.  **Reproject to 3D (Frame $i$):** For a training ray $r_{train}$ corresponding to pixel $p_i$ in frame $i$, compute the 3D point in frame $i$'s camera coordinates $x_{li}$ at scale $l$ using the rendered depth $D_{il}(p_i)$:
    $$ x_{li} = D_{il}(p_i) K_i^{-1} \tilde{p}_i $$
    Alternatively, using the ray origin $o_i(p_i)$ and direction $v_i(p_i)$:
    $$ x_{li} = o_i(p_i) + D_{il}(p_i) v_i(p_i) $$
    Transform $x_{li}$ to world coordinates $x_l$ using frame $i$'s pose $(R_i, t_i)$:
    $$ x_l = R_i x_{li} + t_i $$

2.  **Reproject to Frame $j$:** Transform the world coordinate point $x_l$ to frame $j$'s camera coordinates $x_{li \to j}$ using frame $j$'s pose $(R_j, t_j)$:
    $$ x_{li \to j} = R_j^T (x_l - t_j) $$
    Project $x_{li \to j}$ back to the 2D pixel coordinate system of frame $j$ to get $\tilde{p}_{li \to j}$:
    $$ \tilde{p}_{li \to j} = \pi(K_j x_{li \to j}) $$
    where $\pi([x, y, z]^T) = [x/z, y/z]^T$.

3.  **Compute Reprojection Error:** Calculate the color difference between the original pixel in frame $i$ and the warped pixel in frame $j$:
    $$ e_l(p_i) = C_i(p_i) - C_j(\tilde{p}_{li \to j}) $$
    where $C_i$ and $C_j$ are the RGB maps of frames $i$ and $j$.

4.  **Select Pseudo-Depth:** Identify the scale $l^*$ with the minimal reprojection error for each ray. This scale's depth is treated as the pseudo-ground truth depth $D'(r_{train})$:
    $$ D'(r_{train}) = \arg \min_l (e_l(r_{train})) $$

5.  **Compute Loss:** Calculate the Mean Squared Error (MSE) between the rendered depth at each scale and the selected pseudo-ground truth depth across all training rays $R_{train}$:
    $$ \mathcal{L}_{geo}(r_{train}) = \sum_{l=0}^{L} \sum_{r_{train} \in R_{train}} \| \hat{D}_l(r_{train}) - D'(r_{train}) \|^2 $$
    where $\hat{D}_l$ is the rendered depth at scale $l$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cross_scale_geo_loss(frames, poses, intrinsics, voxels, rays):
    """
    Compute Lgeo for a batch of rays.
    frames: List of RGB images [H, W, 3]
    poses: List of [R, t] for each frame
    intrinsics: K matrix
    voxels: Multi-scale voxel structure providing render_depth and render_color
    rays: List of ray indices or coordinates corresponding to pixels in frame 0
    """
    L_geo_total = 0.0
    num_scales = len(voxels.scales)
    
    # Assume frame 0 is source (i) and frame 1 is target (j) for simplicity
    # In practice, this is computed for all pairs or a subset
    frame_i = frames[0]
    frame_j = frames[1]
    R_i, t_i = poses[0]
    R_j, t_j = poses[1]
    K_i = intrinsics
    K_j = intrinsics
    
    for ray in rays:
        p_i = ray.pixel_coord # 2D coordinate [u, v]
        # Homogeneous coordinate
        p_i_homo = np.array([p_i[0], p_i[1], 1.0])
        
        min_error = float('inf')
        best_scale_idx = 0
        best_depth = 0.0
        
        # Step 1 & 2: Reproject through all scales
        for l in range(num_scales):
            # Get depth from volume rendering at scale l
            D_l = voxels.render_depth(p_i, scale=l)
            
            # Compute 3D point in camera frame i
            # x_li = D_l * K_i^-1 * p_i_homo
            x_li_cam = D_l * np.linalg.inv(K_i) @ p_i_homo
            
            # Transform to world coordinates
            # x_l = R_i * x_li_cam + t_i
            x_l_world = R_i @ x_li_cam + t_i
            
            # Transform to camera frame j
            # x_li_j_cam = R_j^T * (x_l_world - t_j)
            x_li_j_cam = R_j.T @ (x_l_world - t_j)
            
            # Project to 2D in frame j
            # p_j = pi(K_j * x_li_j_cam)
            p_j_homo = K_j @ x_li_j_cam
            p_j = p_j_homo[:2] / p_j_homo[2]
            
            # Get color at projected location in frame j
            # Interpolate C_j at p_j
            C_j_val = voxels.render_color(p_j, scale=l, frame=1)
            C_i_val = frame_i[p_i[1], p_i[0]] # Direct lookup for source
            
            # Compute error
            error = np.linalg.norm(C_i_val - C_j_val)
            
            if error < min_error:
                min_error = error
                best_scale_idx = l
                best_depth = D_l
        
        # Step 5: Accumulate loss against the best depth
        # L_geo = sum( (D_l - D_best)^2 )
        for l in range(num_scales):
            D_l = voxels.render_depth(p_i, scale=l)
            L_geo_total += (D_l - best_depth) ** 2
            
    return L_geo_total
```

**Parameters:**
*   **Number of Scales ($L$):** Typically 3 (Low, Mid, High) [S1].
*   **Loss Weight ($\lambda_{geo}$):** Not explicitly defined in the provided notes, but added to total loss as $\lambda_{reg} \mathcal{L}_{reg}$ [Parent].

**Edge Cases and Pitfalls:**
*   **Sparse Views:** The method relies on accurate camera poses. In scenarios with significant viewpoint changes or sparse training views (e.g., 360° datasets), generalization may suffer [S1].
*   **Local Minima:** Omitting novel view rays risks local minima, causing incorrect geometry [S1].
*   **Scale Mismatch:** Using a single depth prior ($L_d$) can lead to higher error later in training due to scale mismatches, whereas $L_{geo}$ refines depth progressively [S1].

## Subtopics

### Low-Resolution Guidance
Uses low-resolution voxels as pseudo-ground truth early in training to guide coarse geometry learning, suppressing floaters effectively [S1].

### High-Resolution Refinement
Medium- and high-resolution voxels gradually refine details as training advances, adapting to higher frequencies [S1].

### Comparison with Depth Prior ($L_d$)
$L_{geo}$ provides a stronger and more robust constraint than the depth prior, achieving lower final depth error and better convergence quality across scales [S1].

## Operating characteristics
*   **Time Complexity:** Adds overhead for multi-scale rendering and reprojection calculations for each ray. However, it eliminates the need for complex scheduling or ground truth pose sampling [S1].
*   **Memory Complexity:** Requires storing multi-scale voxel representations and intermediate 3D points for reprojection.
*   **Performance:** Significantly reduces floaters and improves convergence quality compared to training without geometric adaptation [S1]. Achieves lower final depth error than $L_d$ alone [S1].
*   **Strengths:** Self-adaptive, no external priors needed, robust to geometric noise compared to feature-warped methods like GeCoNeRF [S1].
*   **Limitations:** Requires accurate camera poses; performance may degrade with very sparse views or large viewpoint changes [S1].

## Background
Cross-scale geometric adaptation was proposed in **FrugalNeRF** (2025) by the authors of [S1]. It was introduced to accelerate and regularize training for extreme few-shot novel view synthesis without learned priors. The method leverages weight-sharing voxels across scales to capture varying scene frequencies and uses reprojection errors to guide multi-scale geometry learning. It addresses the limitations of single-scale depth priors and feature-based warping methods, which are sensitive to geometric noise and floaters [S1].

## Sources
[S1] FrugalNeRF: Fast Convergence for Extreme Few-shot Novel View Synthesis without Learned Priors, https://doi.org/10.1109/cvpr52734.2025.01049, fetched 2026-10-29
