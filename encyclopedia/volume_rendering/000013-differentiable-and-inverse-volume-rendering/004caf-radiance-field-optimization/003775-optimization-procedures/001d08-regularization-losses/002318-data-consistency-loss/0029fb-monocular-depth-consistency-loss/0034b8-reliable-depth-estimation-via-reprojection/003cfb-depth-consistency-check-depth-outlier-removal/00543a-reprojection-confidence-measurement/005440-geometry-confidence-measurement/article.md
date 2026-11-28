# Geometry Confidence Measurement

Also called: Depth and flow consistency

## Overview
A method to measure geometry consistency of depths and flows across different views to identify outliers in multi-view stereo and neural rendering pipelines [S1, unofficial]. It sets itself apart from sibling methods by explicitly combining geometric reprojection errors with optical flow consistency, rather than relying solely on photometric or feature similarity [S1, unofficial]. It builds on [Reprojection Confidence Measurement](15150-reprojection-confidence-measurement.md) by adding a flow-based constraint to detect motion inconsistencies that reprojection alone might miss [G].

## How it works
**Inputs:**
- Source depth image $D_s$ and source image $I_s$ [S1, unofficial].
- Set of target views, each with depth image $D_t$ and image $I_t$ [S1, unofficial].
- Camera poses for source and target views [G].
- Threshold $\tau$ for outlier identification [S1, unofficial].

**Outputs:**
- Depth consistency map $C_{depth}$ [S1, unofficial].
- Flow consistency map $C_{flow}$ [S1, unofficial].
- Combined geometry confidence $\hat{C}$ (optional, via learnable weights) [S1, unofficial].

**Data Structures:**
- Pixel coordinates: $X_s = (x_s, y_s, d_s)$, $X_t = (x_t, y_t, d_t)$ [S1, unofficial].
- Depth values: $d_s$ (source), $d_t$ (reprojected), $\hat{d}_t$ (target view's depth at projected location) [S1, unofficial].
- Flow vectors: $f_{s \to t}$ (computed optical flow), $\Delta_{x,y}$ (reprojection displacement) [S1, unofficial].

**Steps:**
1. **Reprojection:** For each pixel $X_s = (x_s, y_s, d_s)$ in the source depth image $D_s$, project it into the coordinate system of each adjacent target view using the known camera poses [S1, unofficial]. This yields the projected pixel coordinates $X_t = (x_t, y_t)$ and the reprojected depth $d_t$ [S1, unofficial]. Implementers must use standard perspective projection: $X_t = K_t (R_t R_s^T (X_s - t_s) + t_t)$, where $K$ is the intrinsic matrix, and $R$ and $t$ are rotation and translation [G].
2. **Optical Flow Computation:** Compute the pixel motion (optical flow) from the source image to the adjacent target views, denoted as $f_{s \to t}(x_s, y_s)$, using an optical flow method [S1, unofficial].
3. **Depth Consistency Calculation:** Compute the depth consistency $C_{depth}$ for each pixel by comparing the reprojected depth $d_t$ with the original target view's depth $\hat{d}_t = D_t(x_t, y_t)$ [S1, unofficial]. The error is normalized by the source depth $d_s$ [S1, unofficial].
4. **Flow Consistency Calculation:** Compute the flow consistency $C_{flow}$ by comparing the computed optical flow $f_{s \to t}$ with the geometric displacement derived from reprojection $\Delta_{x,y}$ [S1, unofficial]. Define displacement: $\Delta_{x,y} = (x_t - x_s, y_t - y_s)$ [S1, unofficial]. This measures the relative difference between the expected geometric motion and the observed photometric motion [G].
5. **Confidence Combination (Optional):** The final confidence map can be learned by combining multiple metrics (e.g., RGB, SSIM, VGG, Depth, Flow) with learnable weights $\omega_i$ [S1, unofficial]. The weights $\omega$ are optimized jointly during training to automatically focus on correct confidence metrics [S1, unofficial].

**Formulas:**
- Depth consistency:
  $$ C_{depth} = \gamma\left(\frac{|d_t - \hat{d}_t|}{d_s}\right) $$
- Flow consistency:
  $$ C_{flow} = \gamma\left(\frac{\|\Delta_{x,y} - f_{s \to t}(x_s, y_s)\|}{\|\Delta_{x,y}\|}\right) $$
- Clamping function:
  $$ \gamma(x) = \begin{cases} 1 - \frac{x}{\tau} & \text{if } x < \tau \\ 0 & \text{if } x \ge \tau \end{cases} $$
- Combined confidence:
  $$ \hat{C} = \sum_i \omega_i C_i, \quad \text{where } \sum_i \omega_i = 1 $$
All formulas and symbols are defined in the steps above [S1, unofficial].

**Parameters:**
- $\tau$: Threshold for outlier identification. Value not explicitly given in sources, but described as a threshold to identify outliers [S1, unofficial].
- Optical Flow Method: Reference implementation by Zhang et al. (2021) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** In $C_{depth}$, if $d_s = 0$, the normalization fails. Guard against zero or near-zero source depths by clamping $d_s$ to a minimum epsilon or skipping the pixel [G].
- **Zero Displacement:** In $C_{flow}$, if $\|\Delta_{x,y}\| = 0$ (pixel does not move between views), the denominator is zero. This typically occurs for pixels at infinity or static background. Handle by setting $C_{flow} = 1$ (perfect consistency) or ignoring such pixels [G].
- **Occlusions:** Reprojection into occluded regions will yield incorrect $d_t$ or $\hat{d}_t$. The flow consistency helps detect these, but extreme occlusions may still produce misleading confidence [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_geometry_confidence(Ds, Is, Dts, Its, poses, tau, flow_fn):
    """
    Ds: (H, W) source depth
    Is: (H, W, 3) source image
    Dts: list of (H, W) target depths
    Its: list of (H, W, 3) target images
    poses: list of (4, 4) target camera matrices in source camera space
    tau: float, outlier threshold
    flow_fn: function taking (Is, It) -> (H, W, 2) flow
    Returns: C_depth, C_flow, C_combined
    """
    H, W = Ds.shape
    C_depth = np.zeros((H, W))
    C_flow = np.zeros((H, W))
    valid_mask = np.zeros((H, W), dtype=bool)
    
    for t_idx, (Dt, It, pose) in enumerate(zip(Dts, Its, poses)):
        # 1. Reprojection
        ys, xs = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
        Xs = np.stack([xs, ys, Ds], axis=-1).astype(np.float32)
        Xs_hom = np.concatenate([Xs, np.ones_like(Xs[..., :1])], axis=-1)
        Xt_hom = (pose @ Xs_hom.transpose(3, 0, 1, 2)).transpose(1, 2, 3, 0)
        xt = Xt_hom[..., 0] / (Xt_hom[..., 2] + 1e-6)
        yt = Xt_hom[..., 1] / (Xt_hom[..., 2] + 1e-6)
        
        # Interpolate reprojected depth dt from source depth using target poses
        dt = interpolate(Ds, xt, yt)
        # Get target view depth at projected location
        dt_hat = interpolate(Dt, xt, yt)
        
        # 2. Optical Flow
        f_st = flow_fn(Is, It)
        
        # 3. Depth Consistency
        depth_err = np.abs(dt - dt_hat) / (Ds + 1e-6)
        gamma_depth = np.where(depth_err >= tau, 0.0, 1.0 - depth_err / tau)
        C_depth += gamma_depth
        valid_mask |= (np.isfinite(dt) & np.isfinite(dt_hat))
        
        # 4. Flow Consistency
        delta = np.stack([xt - xs, yt - ys], axis=-1)
        delta_norm = np.linalg.norm(delta, axis=-1, keepdims=True) + 1e-6
        flow_err = np.linalg.norm(delta - f_st, axis=-1, keepdims=True) / delta_norm
        gamma_flow = np.where(flow_err >= tau, 0.0, 1.0 - flow_err / tau)
        C_flow += gamma_flow
        valid_mask |= (delta_norm.squeeze() > 1e-3)
        
    # Normalize by number of views
    view_count = np.sum(valid_mask, axis=0) + 1e-6
    C_depth /= view_count
    C_flow /= view_count
    
    # 5. Combine (example weights; optimized during training)
    weights = np.array([0.2, 0.2, 0.2, 0.2, 0.2]) # rgb, ssim, vgg, depth, flow
    C_combined = weights[3] * C_depth + weights[4] * C_flow
    return C_depth, C_flow, C_combined
```

## Subtopics

### [Geometric Consistency Analysis](17624-geometric-consistency-analysis.md)
Geometric Consistency Analysis is a method for evaluating the reliability of predicted depth and opacity in multi-view reconstruction by assigning geometric labels to points based on their agreement with ground-truth (GT) data across multiple views. It solves the problem of distinguishing between accurate surface predictions, free-space errors, and uncertain regions, thereby identifying outliers that violate observed geometry. It sets itself apart from sibling methods like [Depth-free motion evidence (Epipolar Residual)](20194-depth-free-motion-evidence-epipolar-residual.md) by relying on explicit ground-truth depth comparisons rather than motion or epipolar constraints.

### [Depth-free motion evidence (Epipolar Residual)](20194-depth-free-motion-evidence-epipolar-residual.md)
A method to detect motion without requiring correct depth, by measuring the distance of measured correspondences to their epipolar lines in the target frame. It measures motion evidence by calculating the distance from each measured correspondence to its epipolar line in pixels, using no correct depth, only camera pose, making it robust against noisy monocular depth that defeats rigid-flow residuals. This approach serves as a geometry confidence metric within the broader framework of [Geometry Confidence Measurement](15151-geometry-confidence-measurement.md), distinguishing true motion from depth estimation errors by leveraging the geometric constraint that static points must lie on their epipolar lines regardless of depth.

## Operating characteristics
- **Time Complexity:** Depends on the number of target views and the cost of the optical flow method. Optical flow is typically $O(W \cdot H)$ per view pair [G].
- **Memory Complexity:** Requires storing depth maps, flow fields, and confidence maps for all views [G].
- **Strengths:** Explicitly models geometric and photometric consistency, robust to outliers through the $\gamma$ function. The learnable weights $\omega$ adapt the model to automatically focus on correct confidence metrics [S1, unofficial].
- **Limitations:** Requires accurate camera poses and reliable optical flow. Sensitive to the threshold $\tau$. Does not account for textureless regions where flow is unreliable [G].
- **When to Use:** Multi-view stereo, neural rendering, and depth estimation pipelines where outlier removal is critical [S1, unofficial].
- **When Not to Use:** Single-view reconstruction, scenes with significant non-rigid motion (unless flow is robust), or real-time applications where optical flow is too expensive [G].

## Background
The method leverages the redundancy of multi-view observations to enforce geometric consistency [S1, unofficial]. It is used in multi-view reconstruction and neural rendering pipelines to construct robust confidence measurements that expand from pixels to local and non-local regions [S1, unofficial].

## Sources
[S1] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
[S2] Representing 3D Shapes with Probabilistic Directed Distance Fields, https://doi.org/10.1109/cvpr52688.2022.01874, fetched 2026-10-26
