# Depth-Guided Optimization

Also called: Depth-Guided Optimization with 3D Point Clouds

## Overview
Depth-Guided Optimization is a strategy that uses sparse Structure-from-Motion (SfM) and dense PatchMatch Stereo point clouds to guide depth prediction in few-shot novel view synthesis, preventing overfitting to training views [S1]. It differs from standard depth-supervised methods by computing adaptive weights based on multiview color consistency to mitigate inaccuracies in estimated 3D points, and by restricting its application to early training stages [S1].

## How it works
**Inputs:**
- Reference images $I_k$ ($k=1 \dots M$) [S1].
- Camera intrinsics $K_k$ and extrinsics $E_k$ (world-to-camera) for each reference view [S1].
- A hybrid representation model (e.g., TriDF) parameterized by density fields $\sigma$ and features $f_m$ [S1].
- A set of 3D points $P = \{p_i\}$ derived from SfM and PatchMatch Stereo [S1].
- Combine or select points for optimization [S1].
- For each point $p_i$, compute its average color $c_i$ across all $M$ views where it is visible [S1].

**Outputs:**
- Depth loss term $L_{depth}$ used in the total optimization objective [S1].

**Data Structures:**
- **3D Point Cloud:** A list of 3D coordinates $p_i = [X_w, Y_w, Z_w, 1]^T$ in world space [S1].
    - Sparse SfM cloud: $\approx 10^2$ points [S1].
    - Dense Stereo cloud: $\approx 10^3$ points [S1].
- **Keypoint Data:** For each 3D point $p_i$ and view $k$, store projected pixel coordinates $(u, v)$ and the rendered depth $\hat{d}_i$ [S1].

**Parameters:**
- $N_s$: Number of key points sampled per iteration (64) [S1].
- $\lambda_1$: Depth loss weight. Initial: 0.001; Later: 0 [S1].
- $\lambda_2$: Smoothness loss weight. Initial: 0; Later: 1.0 [S1].
- Training Schedule: Depth guidance active for the first 10,000 iterations of a 30,000-iteration process [S1].

**Steps:**

1. **Point Cloud Generation:**
   - Estimate sparse 3D points using SfM (e.g., via COLMAP) [S1].
   - Generate dense 3D points using PatchMatch Stereo and COLMAP fusion [S1].
   - Combine or select points for optimization. The method uses approximately $10^3$ dense points for guidance [S1].

2. **Projection and Rendering:**
   - For each 3D point $p_i$ in the cloud:
     - Transform to camera coordinates for view $k$:
       $$
       \begin{bmatrix} X_c \\ Y_c \\ Z_c \\ 1 \end{bmatrix} = E_k p_i = \begin{bmatrix} R & T \\ 0 & 1 \end{bmatrix} \begin{bmatrix} X_w \\ Y_w \\ Z_w \\ 1 \end{bmatrix}
       $$
       where $R \in \mathbb{R}^{3 \times 3}$ and $T \in \mathbb{R}^{3 \times 1}$ [S1].
     - Project to image plane to get pixel coordinates $(u, v)$:
       $$
       \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} \propto K_k \begin{bmatrix} X_c \\ Y_c \\ Z_c \end{bmatrix} = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix} \begin{bmatrix} X_c \\ Y_c \\ Z_c \end{bmatrix}
       $$
       where $Z_c$ is the depth value [S1].
     - Render predicted depth $\hat{d}_i$ at this location using volume rendering:
       $$
       \hat{d}_i = \sum_{j=1}^{N} T_j (1 - \exp(-\sigma_j \delta_j)) t_j
       $$
       where $t_j$ is the position of the $j$-th sample, $\sigma_j$ is density, $\delta_j$ is distance, and $T_j$ is transmittance [S1].

3. **Error and Weight Calculation:**
   - For each point $p_i$, compute its average color $c_i$ across all $M$ views where it is visible. Let $c_{ij}$ be the color of point $i$ projected on view $j$, and $p_i$ be the color of the point cloud itself [S1].
   - Compute the color difference metric $S(c_a, c_b)$:
     $$
     S(c_a, c_b) = \frac{1}{3} \| c_a - c_b \|_1
     $$
   - Calculate the error $e_{ik}$ for point $i$ on view $k$:
     $$
     e_{ik} = \left( \frac{1}{M-1} \sum_{j=1}^{M} S(c_{ij}, c_i) + S(c_{ik}, p_i) \right)^{1/2}
     $$
     This quantifies multiview inconsistency and edge sensitivity [S1].
   - Compute adaptive weight $\alpha_{ki}$:
     $$
     \alpha_{ki} = 1 - e_{ik}
     $$
   - Clamp the weight:
     $$
     w_{ki} = \text{clamp}(\alpha_{ki}, 0, 1)
     $$
     [S1]

4. **Loss Computation:**
   - Compute the weighted depth loss $L_{depth}$:
     $$
     L_{depth} = \sqrt{ \frac{1}{N_s} \sum_{i=1}^{N_s} w_i (\hat{d}_i - d_i)^2 }
     $$
     where $d_i$ is the depth of the 3D point cloud (from stereo/SfM) and $w_i$ is the computed weight [S1].

5. **Integration into Training:**
   - The total loss is:
     $$
     L_{total} = L_{color} + \lambda_1 L_{depth} + \lambda_2 L_{smooth}
     $$
   - During the first 10,000 iterations, set $\lambda_1 = 0.001$ and $\lambda_2 = 0$ [S1].
   - After 10,000 iterations, set $\lambda_1 = 0$ and $\lambda_2 = 1.0$ to switch to edge-aware smoothness regularization [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def l1_norm(a, b):
    return sum(abs(x - y) for x, y in zip(a, b))

def color_dist(c1, c2):
    return (1.0 / 3.0) * l1_norm(c1, c2)

def sample_color(image, u, v):
    h, w, _ = image.shape
    u_int = max(0, min(w - 1, int(round(u))))
    v_int = max(0, min(h - 1, int(round(v))))
    return image[v_int, u_int]

def compute_depth_guided_loss(model, points_3d, camera_params, reference_images, iteration):
    """
    Computes the depth-guided loss for early training stages.
    model: Neural network providing render_depth(u, v) [G].
    points_3d: List of 3D points [X, Y, Z, 1] in world coordinates.
    camera_params: List of dicts with 'K' (3x3) and 'E' (4x4) for each view.
    reference_images: List of HxWx3 RGB arrays.
    iteration: Current training iteration number.
    Returns: Scalar depth loss L_depth.
    """
    if iteration >= 10000:
        return 0.0

    M = len(reference_images)
    N_s = 64
    total_weighted_error = 0.0
    count = 0

    for p_i in points_3d[:N_s]:
        visible_colors = []
        Zc = 0.0

        for k in range(M):
            K = camera_params[k]['K']
            E = camera_params[k]['E']

            # Transform to camera coordinates
            p_cam = [sum(E[r][c] * p_i[c] for c in range(4)) for r in range(4)]
            Xc, Yc, Zc = p_cam[0], p_cam[1], p_cam[2]

            # Project to image plane
            uv_hom = [sum(K[r][c] * (Xc if c < 3 else 1) for c in range(3)) for r in range(3)]
            u = uv_hom[0] / uv_hom[2]
            v = uv_hom[1] / uv_hom[2]

            if 0 <= u < reference_images[k].shape[1] and 0 <= v < reference_images[k].shape[0]:
                visible_colors.append(sample_color(reference_images[k], u, v))

        if len(visible_colors) < 2:
            continue

        # Average color across visible views
        c_i = [sum(c[j] for c in visible_colors) / len(visible_colors) for j in range(3)]

        # Color of the point cloud itself [G]
        p_i_color = visible_colors[0]

        # Compute error e_i (averaged over visible views to match loss formula)
        sum_S = sum(color_dist(c_vis, c_i) for c_vis in visible_colors)
        term1 = sum_S / (len(visible_colors) - 1)
        term2 = color_dist(visible_colors[0], p_i_color)
        e_i = math.sqrt(term1 + term2)

        # Weight
        alpha_i = 1.0 - e_i
        w_i = max(0.0, min(1.0, alpha_i))

        # Render predicted depth at this location
        d_hat = model.render_depth(u, v)

        # Depth loss contribution
        d_i = Zc
        total_weighted_error += w_i * (d_hat - d_i) ** 2
        count += 1

    if count == 0:
        return 0.0

    return math.sqrt(total_weighted_error / count)
```

## Operating characteristics
- **Time Complexity:** Adds overhead per iteration for projecting points, rendering depth at specific locations, and computing multiview color consistency. The projection and rendering are done for $N_s$ points (64) per iteration [S1].
- **Memory Complexity:** Requires storing the 3D point cloud ($\approx 10^3$ points) and reference image features [S1].
- **Performance:** Improves geometric accuracy in few-shot settings by preventing overfitting to training views, which manifests as excessive density near the ray origin [S1].
- **Limitations:** Relies on the accuracy of SfM and PatchMatch Stereo. Inaccuracies in point clouds (spatial position errors, edge ambiguities) can compromise geometry if not weighted correctly [S1].
- **When to use:** Early training stages (first 10k iterations) of few-shot novel view synthesis where depth guidance is needed to anchor geometry [S1].
- **When not to use:** Later training stages, where it is replaced by edge-aware smoothness regularization to refine geometry without the noise of potentially inaccurate point clouds [S1].

## Background
Depth-Guided Optimization was introduced in the context of TriDF (Triplane-Accelerated Density Fields) for few-shot remote sensing novel view synthesis [S1]. It addresses the issue of overfitting in hybrid representations (combining implicit density fields with explicit features) when training data is limited [S1]. The method leverages the fact that SfM is often already performed for camera pose estimation, adding minimal computational overhead [S1]. It was developed to handle the sparse nature of SfM points by augmenting them with dense PatchMatch Stereo points generated via COLMAP [S1].

## Sources
[S1] TriDF: Triplane-Accelerated Density Fields for Few-Shot Remote Sensing Novel View Synthesis, https://doi.org/10.1109/tgrs.2025.3643432, fetched 2026-10-30
