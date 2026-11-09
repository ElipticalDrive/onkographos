# HexPlane


## Overview
HexPlane is an explicit representation for dynamic 3D scenes that decomposes a 4D spacetime grid into six feature planes spanning each pair of coordinate axes (e.g., XY, ZT) [S1]. It computes features for points in spacetime by projecting the point onto each feature plane, aggregating the six resulting feature vectors, and passing the fused vector to a tiny MLP to predict color [S1]. This method solves the memory and data-sparsity challenges of dynamic novel view synthesis by reducing memory complexity from $O(N^3TF)$ to $O(N^2R + NTR + RF)$ while matching the image quality of implicit methods but with over 100× faster training [S1]. Unlike implicit NeRF variants or deformation-based methods, HexPlane uses a basis-sharing mechanism to cope with sparse observations without requiring deformation fields [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed and timestamped images of a dynamic scene.
*   **Output:** A differentiable function mapping a 3D position $\mathbf{x}=(x,y,z)$, time $t$, and viewing direction $\mathbf{d}$ to an RGB color $\mathbf{c}$ and opacity $\alpha$.

**Data Structures:**
The representation consists of two separate HexPlane structures: one for opacity and one for appearance features. Each HexPlane is defined by three pairs of feature planes.
*   **Feature Planes:** Six learned tensors (planes) per HexPlane.
    *   Pair 1: $M^{XY} \in \mathbb{R}^{X \times Y}$ and $M^{ZT} \in \mathbb{R}^{Z \times T}$.
    *   Pair 2: $M^{XZ} \in \mathbb{R}^{X \times Z}$ and $M^{YT} \in \mathbb{R}^{Y \times T}$.
    *   Pair 3: $M^{YZ} \in \mathbb{R}^{Y \times Z}$ and $M^{XT} \in \mathbb{R}^{X \times T}$.
    *   Here $X, Y, Z$ are spatial grid resolutions and $T$ is the temporal resolution.
*   **Basis Vectors:** Three learned matrices $V^{R_1 F}, V^{R_2 F}, V^{R_3 F}$ where $F$ is the feature dimension (e.g., 3 for RGB) and $R_i$ is the number of basis components for pair $i$.
*   **Emptiness Voxel:** A tiny 3D voxel grid indicating empty regions, computed by taking the maximum opacity across all time steps.
*   **MLP:** A tiny Multi-Layer Perceptron that takes the fused appearance feature, point coordinates, and view direction to regress RGB color.

**Parameters:**
*   **Spatial Resolution ($N$):** Grid size for X, Y, Z axes. Typical values: $128^3$ to $512^3$ [S1].
*   **Temporal Resolution ($T$):** Number of timesteps. Typical value: 32 or 300 [S1].
*   **Basis Number ($R$):** Number of shared volumes/planes. Typical values: 8, 12, 16, 24, 48, 96. Higher $R$ improves quality but increases computation and risk of overfitting [S1].
    *   Plenoptic Video: Appearance $R_1=48, R_2=24, R_3=24$; Opacity $R_1=24, R_2=12, R_3=12$ [S1].
    *   D-NeRF: Appearance $R_1=R_2=R_3=48$; Opacity $R_1=R_2=R_3=24$ [S1].
*   **Coordinate System:** Normalized Device Coordinates (NDC) for bounded scenes [S1]. For unbounded scenes, spherical coordinates $(\theta, \phi, r, t)$ are used where $r = 1/\sqrt{x^2+y^2+z^2}$ [S1].

**Query Function:**
The HexPlane representation $D$ for a 4D feature volume $V \in \mathbb{R}^{XYZT F}$ is defined as [S1]:
$$
D(x, y, z, t) = (\mathbf{P}_{xy\bullet}^{XYR1} \odot \mathbf{P}_{xt\bullet}^{ZTR1}) \mathbf{V}^{R1F} + (\mathbf{P}_{xz\bullet}^{XZR2} \odot \mathbf{P}_{yt\bullet}^{YTR2}) \mathbf{V}^{R2F} + (\mathbf{P}_{yz\bullet}^{YZR3} \odot \mathbf{P}_{xt\bullet}^{XTR3}) \mathbf{V}^{R3F}
$$
where $\odot$ is element-wise product, $\mathbf{P}$ tensors stack planes, $\mathbf{V}$ tensors stack vectors, and subscripts denote bilinear interpolation [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize the six feature planes for the opacity HexPlane and the six feature planes for the appearance HexPlane. Initialize the emptiness voxel grid. Initialize the tiny MLP.
2.  **Ray Sampling:** For each ray $\mathbf{r}$, sample points $\mathbf{x}_i = (x_i, y_i, z_i, t_i)$ along the ray. If using spherical coordinates for unbounded scenes, sample $r$ linearly between 0 and 1 [S1].
3.  **Opacity Query:** Query the opacity HexPlane at $(x_i, y_i, z_i, t_i)$ to get a scalar opacity value.
4.  **Emptiness Check:** If the point falls within an empty voxel region (based on the emptiness voxel grid), skip rendering for this point [S1].
5.  **Appearance Feature Query:** If the opacity is high (threshold not specified, [G]), query the appearance HexPlane at $(x_i, y_i, z_i, t_i)$ to get a feature vector $\mathbf{f}_i \in \mathbb{R}^F$ [S1].
6.  **Color Regression:** Pass $\mathbf{f}_i$, the point coordinates $(x_i, y_i, z_i)$, and the viewing direction $\mathbf{d}$ to the tiny MLP to regress the RGB color $\mathbf{c}_i$ [S1].
7.  **Volumetric Rendering:** Compute the pixel color $C(\mathbf{r})$ using standard volumetric rendering:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\sigma_i$ is the opacity from step 3 and $\delta_i$ is the distance between samples [S1].
8.  **Loss Calculation:** Compute the photometric loss between the rendered image and the ground-truth image:
    $$
    \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \|C(\mathbf{r}) - \hat{C}(\mathbf{r})\|_2^2 + \lambda_{\text{reg}} \mathcal{L}_{\text{reg}}
    $$
    where $\hat{C}(\mathbf{r})$ is the ground truth color and $\mathcal{L}_{\text{reg}}$ is the regularization loss [S1].
9.  **Regularization:** Apply Total Variational (TV) loss on all feature planes to enforce spatial-temporal continuity. Apply depth smooth loss to reduce artifacts [S1].
10. **Optimization:** Update the HexPlane planes, basis vectors, and MLP parameters using an optimizer (e.g., Adam) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bilinear_interpolate(plane, u, v):
    """
    Perform bilinear interpolation on a 2D plane at normalized coordinates u, v.
    plane: shape (H, W)
    u, v: shape (N,) normalized coordinates in [0, 1]
    returns: shape (N,)
    """
    H, W = plane.shape
    # Map normalized coords to pixel indices
    u_idx = u * (W - 1)
    v_idx = v * (H - 1)
    
    # Floor and ceil indices
    u0 = np.floor(u_idx).astype(int)
    u1 = np.minimum(u0 + 1, W - 1)
    v0 = np.floor(v_idx).astype(int)
    v1 = np.minimum(v0 + 1, H - 1)
    
    # Weights
    w_u = u_idx - u0
    w_v = v_idx - v0
    
    # Interpolate
    val = (1 - w_u) * (1 - w_v) * plane[v0, u0] + \
          (1 - w_u) * w_v * plane[v1, u0] + \
          w_u * (1 - w_v) * plane[v0, u1] + \
          w_u * w_v * plane[v1, u1]
    return val

def query_hexplane(opacity_planes, appearance_planes, V_matrices, x, y, z, t, R1, R2, R3):
    """
    Query the HexPlane representation.
    opacity_planes: dict with keys 'XY', 'ZT', 'XZ', 'YT', 'YZ', 'XT'
    appearance_planes: dict with keys 'XY', 'ZT', 'XZ', 'YT', 'YZ', 'XT'
    V_matrices: list of [V1, V2, V3] where Vi is shape (Ri, F)
    x, y, z, t: arrays of shape (N,) normalized coordinates in [0, 1]
    R1, R2, R3: int, number of basis components for each pair
    returns: opacity (N,), appearance_features (N, F)
    """
    # 1. Query Opacity HexPlane
    # Pair 1: XY and ZT
    p_xy_op = bilinear_interpolate(opacity_planes['XY'], x, y) # Shape (N, R1) if stacked, but planes are single channel?
    # Note: The paper defines M_r as a plane. P_xy stacks M_r.
    # In implementation, we usually store planes as (H, W, R) or query per R.
    # Assuming planes are stored as (H, W, R) for simplicity in this pseudocode,
    # or we iterate R. The formula uses P_xy^{XYR1} which implies stacking.
    
    # Let's assume planes are stored as (H, W, R)
    # Extract features for all R1 components
    # p_xy_op: (N, R1)
    p_xy_op = np.stack([bilinear_interpolate(opacity_planes['XY'][:,:,r], x, y) for r in range(R1)], axis=1)
    p_zt_op = np.stack([bilinear_interpolate(opacity_planes['ZT'][:,:,r], z, t) for r in range(R1)], axis=1)
    
    # Element-wise product
    prod1_op = p_xy_op * p_zt_op # (N, R1)
    
    # Pair 2: XZ and YT
    p_xz_op = np.stack([bilinear_interpolate(opacity_planes['XZ'][:,:,r], x, z) for r in range(R2)], axis=1)
    p_yt_op = np.stack([bilinear_interpolate(opacity_planes['YT'][:,:,r], y, t) for r in range(R2)], axis=1)
    prod2_op = p_xz_op * p_yt_op # (N, R2)
    
    # Pair 3: YZ and XT
    p_yz_op = np.stack([bilinear_interpolate(opacity_planes['YZ'][:,:,r], y, z) for r in range(R3)], axis=1)
    p_xt_op = np.stack([bilinear_interpolate(opacity_planes['XT'][:,:,r], x, t) for r in range(R3)], axis=1)
    prod3_op = p_yz_op * p_xt_op # (N, R3)
    
    # Concatenate and multiply by V
    # V matrices: V1 (R1, F), V2 (R2, F), V3 (R3, F)
    # prod1_op @ V1.T -> (N, F)
    opacity = prod1_op @ V_matrices[0].T + prod2_op @ V_matrices[1].T + prod3_op @ V_matrices[2].T
    
    # 2. Query Appearance HexPlane (only if opacity is high)
    # For simplicity, we query all and mask later, or query conditionally.
    # The paper says "query appearance features of points with high opacities".
    # We will compute appearance features for all points here for the sake of the function,
    # but in practice, you might skip this step for low-opacity points.
    
    p_xy_ap = np.stack([bilinear_interpolate(appearance_planes['XY'][:,:,r], x, y) for r in range(R1)], axis=1)
    p_zt_ap = np.stack([bilinear_interpolate(appearance_planes['ZT'][:,:,r], z, t) for r in range(R1)], axis=1)
    prod1_ap = p_xy_ap * p_zt_ap
    
    p_xz_ap = np.stack([bilinear_interpolate(appearance_planes['XZ'][:,:,r], x, z) for r in range(R2)], axis=1)
    p_yt_ap = np.stack([bilinear_interpolate(appearance_planes['YT'][:,:,r], y, t) for r in range(R2)], axis=1)
    prod2_ap = p_xz_ap * p_yt_ap
    
    p_yz_ap = np.stack([bilinear_interpolate(appearance_planes['YZ'][:,:,r], y, z) for r in range(R3)], axis=1)
    p_xt_ap = np.stack([bilinear_interpolate(appearance_planes['XT'][:,:,r], x, t) for r in range(R3)], axis=1)
    prod3_ap = p_yz_ap * p_xt_ap
    
    appearance_features = prod1_ap @ V_matrices[0].T + prod2_ap @ V_matrices[1].T + prod3_ap @ V_matrices[2].T
    
    return opacity, appearance_features

def render_hexplane(opacity_planes, appearance_planes, V_matrices, mlp, rays, emptiness_voxel, R1, R2, R3):
    """
    Render images from HexPlane.
    rays: list of rays, each ray is (origin, direction, t_start, t_end, num_samples)
    emptiness_voxel: 3D grid (H, W, D) indicating empty regions
    returns: rendered images
    """
    # 1. Sample points along rays
    # 2. Check emptiness
    # 3. Query Opacity
    # 4. Query Appearance (if opacity high)
    # 5. Regress Color via MLP
    # 6. Volume Render
    pass

def train_hexplane():
    """
    Training loop for HexPlane.
    """
    # 1. Initialize HexPlane planes, V matrices, MLP, emptiness voxel
    # 2. Coarse to Fine Training: Start with low resolution, gradually increase
    # 3. For each iteration:
    #    a. Sample rays
    #    b. Render
    #    c. Compute Photometric Loss
    #    d. Compute Regularization (TV loss on planes, depth smooth loss)
    #    e. Backprop and Update
    # 4. Update Emptiness Voxel periodically (e.g., every 50k iterations)
    pass
```

**Edge Cases and Pitfalls:**
*   **Out-of-Boundaries:** HexPlane is limited to bounded scenes because grid sampling fails for out-of-boundary points. NDC requires bounded x, y values and faces forward-facing assumptions, leading to artifacts and incorrect geometries at extreme views [S1]. Use spherical coordinates for unbounded scenes [S1].
*   **Fast Motion/Sparse Observations:** HexPlane generates degraded results when objects move too fast or there are too few observations to synthesize details. The basis-sharing mechanism is less effective than deformation fields for scenes with extremely sparse observations [S1].
*   **Artifacts:** Explicit representations like HexPlane are prone to artifacts and color jittering, requiring strong regularizations (TV loss, depth smooth loss) [S1].
*   **Overfitting:** An unsuitable large basis number $R$ may lead to overfitting, harming synthesis quality on novel views [S1].

## Operating characteristics
*   **Time Complexity:** Querying a HexPlane requires six bilinear interpolations and one vector-matrix product per term (three terms), making it highly efficient compared to MLP evaluations [S1].
*   **Memory Complexity:** Memory usage is reduced to $O(N^2 R + N T R + R F)$, where $N$ is spatial resolution, $T$ is temporal resolution, $R$ is rank, and $F$ is feature dimension [S1]. This is a significant reduction from the naive $O(N^3 T F)$ [S1].
*   **Measured Performance:**
    *   On the Plenoptic Video dataset, HexPlane (650k iterations) achieves 29.470 PSNR, 0.018 D-SSIM, 0.078 LPIPS, 8.16 JOD in 12 GPU hours (112x speedup over DyNeRF) [S1].
    *   HexPlane† (100k iterations) achieves 29.263 PSNR, 0.020 D-SSIM, 0.097 LPIPS, 8.14 JOD in 2 GPU hours (672x speedup over DyNeRF) [S1].
    *   On the D-NeRF dataset, HexPlane achieves 31.04 PSNR, 0.97 SSIM, 0.04 LPIPS in 11 minutes training time [S1].
*   **Model Size:** HexPlane model size is 200MB [S1].
*   **Strengths:** High efficiency, low memory footprint, no need for deformation fields, robust to different fusion mechanisms and coordinate systems [S1].
*   **Limitations:** Limited to bounded scenes (unless spherical coordinates are used), prone to artifacts requiring strong regularization, less effective than deformation fields for extremely sparse observations [S1].
*   **When to Use:** When fast training and low memory usage are critical for dynamic novel view synthesis, and the scene is reasonably well-observed and bounded.
*   **When Not to Use:** For scenes with extremely sparse observations where deformation fields are more effective, or for unbounded scenes without spherical coordinate adaptation [S1].

## Background
HexPlane was introduced by Ang Cao and Justin Johnson from the University of Michigan, Ann Arbor, in 2023 [S1]. It builds on the idea of factored representations for static scenes (like TensoRF) and extends it to 4D spacetime by decomposing the volume into six feature planes [S1]. The method addresses the limitations of implicit representations (slow training due to MLP evaluations) and naive 4D grids (high memory usage) [S1]. It is most closely related to K*-Planes, which also employs six feature planes [S1].

## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
