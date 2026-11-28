# Geometry Confidence Measurement

Also called: Depth and flow consistency

## Overview
A method to measure geometry consistency of depths and flows across different views to identify outliers in multi-view stereo and neural rendering pipelines [S1, unofficial]. It sets itself apart from sibling methods by explicitly combining geometric reprojection errors with optical flow consistency, rather than relying solely on photometric or feature similarity [S1, unofficial]. It builds on [Reprojection Confidence Measurement](15150-reprojection-confidence-measurement.md) by adding a flow-based constraint to detect motion inconsistencies that reprojection alone might miss.

## How it works
**Inputs:**
- Source depth image $D_s$ and source image $I_s$.
- Set of target views, each with depth image $D_t$ and image $I_t$.
- Camera poses for source and target views.
- Threshold $\tau$ for outlier identification.

**Outputs:**
- Depth consistency map $C_{depth}$.
- Flow consistency map $C_{flow}$.
- Combined geometry confidence (optional, via learnable weights).

**Data Structures:**
- Pixel coordinates: $X_s = (x_s, y_s, d_s)$, $X_t = (x_t, y_t, d_t)$.
- Depth values: $d_s$ (source), $d_t$ (reprojected), $\hat{d}_t$ (target ground truth/reconstruction).
- Flow vectors: $f_{s \to t}$ (computed optical flow), $\Delta_{x,y}$ (reprojection displacement).

**Steps:**
1. **Reprojection:** For each pixel $X_s = (x_s, y_s, d_s)$ in the source depth image $D_s$, project it into the coordinate system of each adjacent target view using the known camera poses. This yields the projected pixel coordinates $X_t = (x_t, y_t)$ and the reprojected depth $d_t$.
   - *Note:* The specific projection equation is referenced as "equation 4" in the source [S1, unofficial]. Implementers must use standard perspective projection: $X_t = K_t (R_t R_s^T (X_s - t_s) + t_t)$, where $K$ is the intrinsic matrix, $R$ and $t$ are rotation and translation.
2. **Optical Flow Computation:** Compute the pixel motion (optical flow) from the source image to the adjacent target views, denoted as $f_{s \to t}(x_s, y_s)$, using an optical flow method [S1, unofficial]. The source cites Zhang et al. (2021) as the reference implementation.
3. **Depth Consistency Calculation:** Compute the depth consistency $C_{depth}$ for each pixel by comparing the reprojected depth $d_t$ with the original target view's depth $\hat{d}_t = D_t(x_t, y_t)$.
   - Formula:
     $$ C_{depth} = \gamma\left(\frac{|d_t - \hat{d}_t|}{d_s}\right) $$
   - Where $\gamma(x)$ is a clamping function:
     $$ \gamma(x) = \begin{cases} 1 - \frac{x}{\tau} & \text{if } x < \tau \\ 0 & \text{if } x \ge \tau \end{cases} $$
   - Here, $\tau$ is a threshold parameter used to identify outliers. The error is normalized by the source depth $d_s$.
4. **Flow Consistency Calculation:** Compute the flow consistency $C_{flow}$ by comparing the computed optical flow $f_{s \to t}$ with the geometric displacement derived from reprojection $\Delta_{x,y}$.
   - Define displacement: $\Delta_{x,y} = (x_t - x_s, y_t - y_s)$.
   - Formula:
     $$ C_{flow} = \gamma\left(\frac{\|\Delta_{x,y} - f_{s \to t}(x_s, y_s)\|}{\|\Delta_{x,y}\|}\right) $$
   - This measures the relative difference between the expected geometric motion and the observed photometric motion.
5. **Confidence Combination (Optional):** The final confidence map can be learned by combining multiple metrics (e.g., RGB, SSIM, VGG, Depth, Flow) with learnable weights $\omega_i$:
   $$ \hat{C} = \sum_i \omega_i C_i, \quad \text{where } \sum_i \omega_i = 1 $$
   - The weights $\omega$ are optimized jointly during training to automatically focus on correct confidence metrics [S1, unofficial].

**Parameters:**
- $\tau$: Threshold for outlier identification. Value not explicitly given in sources, but described as a threshold to identify outliers [S1, unofficial].
- Optical Flow Method: Zhang et al. (2021) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** In $C_{depth}$, if $d_s = 0$, the normalization fails. Guard against zero or near-zero source depths.
- **Zero Displacement:** In $C_{flow}$, if $\|\Delta_{x,y}\| = 0$ (pixel does not move between views), the denominator is zero. This typically occurs for pixels at infinity or static background. Handle by setting $C_{flow} = 1$ (perfect consistency) or ignoring such pixels.
- **Occlusions:** Reprojection into occluded regions will yield incorrect $d_t$ or $\hat{d}_t$. The flow consistency helps detect these, but extreme occlusions may still produce misleading confidence.

## Subtopics

### Geometric Consistency Analysis
A variant that assigns geometric labels to points independently of predicted opacity values, using ground-truth depth as the reference [S2, unofficial]. It excludes samples located at depth edges from the analysis. A point is classified as **geometrically inconsistent** if it violates GT-observed free space in at least two views AND agrees with no observed surface depth. A point is **surface-consistent** if it agrees with at least two views AND has no free-space violations. All remaining points are classified as **uncertain**. The depth tolerance threshold is $\max(0.03 \text{ m}, 0.01d)$, where $d$ is the depth value. On held-out Hypersim validation scenes, this procedure identifies 36,125 inconsistent points [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of target views and the cost of the optical flow method. Optical flow is typically $O(W \cdot H)$ per view pair.
- **Memory Complexity:** Requires storing depth maps, flow fields, and confidence maps for all views.
- **Strengths:** Explicitly models geometric and photometric consistency, robust to outliers through the $\gamma$ function. Learnable combination allows adaptation to different scene types [S1, unofficial].
- **Limitations:** Requires accurate camera poses and reliable optical flow. Sensitive to the threshold $\tau$. Does not account for textureless regions where flow is unreliable.
- **When to Use:** Multi-view stereo, neural rendering, and depth estimation pipelines where outlier removal is critical.
- **When Not to Use:** Single-view reconstruction, scenes with significant non-rigid motion (unless flow is robust), or real-time applications where optical flow is too expensive.

## Background
Originated in the context of street view neural radiance fields, specifically S-NeRF [S1, unofficial]. The method leverages the redundancy of multi-view observations to enforce geometric consistency. The geometric consistency analysis variant was further developed in InfiniSplat v2 for feed-forward 3D Gaussian Splatting [S2, unofficial].

## Sources
[S1] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
[S2] InfiniSplat v2: Analyzing and Improving Feed-Forward 3DGS from Any Views, https://arxiv.org/abs/2610.20341, fetched 2026-10-29 (unofficial)
