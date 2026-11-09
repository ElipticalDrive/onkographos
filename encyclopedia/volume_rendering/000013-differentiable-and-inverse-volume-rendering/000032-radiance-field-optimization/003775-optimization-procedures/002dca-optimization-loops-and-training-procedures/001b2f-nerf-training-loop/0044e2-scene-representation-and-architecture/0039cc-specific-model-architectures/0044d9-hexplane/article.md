# HexPlane


## Overview
HexPlane is an explicit representation for dynamic 3D scenes that decomposes a 4D spacetime grid into six feature planes spanning each pair of coordinate axes (e.g., XY, ZT) [S1]. It computes features for points in spacetime by projecting the point onto each feature plane, aggregating the six resulting feature vectors, and passing the fused vector to a tiny MLP to predict color [S1]. It sets itself apart from implicit methods (like T-NeRF or deformation-based NeRFs) by avoiding large MLP evaluations and deformation fields, achieving comparable or better synthesis quality with over 100× acceleration [S1]. It differs from concurrent works using time-aware MLPs by being a more elegant explicit representation without time-aware MLPs [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed and timestamped images of a dynamic scene.
*   **Outputs:** A model capable of rendering novel views at arbitrary poses and times via differentiable volumetric rendering.

**Data Structures:**
The HexPlane representation $D$ for a 4D feature volume $V \in \mathbb{R}^{XYZT F}$ is defined as the sum of three terms, each combining two spatial feature planes and one temporal vector [S1]:

$$
D = \sum_{r=1}^{R_1} M_r^{XY} \circ M_r^{ZT} \circ v_r^1 + \sum_{r=1}^{R_2} M_r^{XZ} \circ M_r^{YT} \circ v_r^2 + \sum_{r=1}^{R_3} M_r^{YZ} \circ M_r^{XT} \circ v_r^3
$$

where:
*   $M_r^{AB} \in \mathbb{R}^{AB}$ are learned feature planes (matrices) spanning axes $A$ and $B$.
*   $v_r^i \in \mathbb{R}^F$ are learned feature vectors along the feature dimension.
*   $\circ$ denotes the outer product.
*   $R_1, R_2, R_3$ are the number of low-rank components (basis numbers) for each term.
*   The representation consists of six feature planes: $P_{XY}, P_{XZ}, P_{YZ}$ (spatial) and $P_{ZT}, P_{YT}, P_{XT}$ (spatio-temporal) [S1].

**Query Function:**
The query function $D(x, y, z, t)$ mapping a point to an $F$-dimensional feature vector is [S1]:

$$
D(x, y, z, t) = (P_{xy\bullet}^{XYR1} \odot P_{xt\bullet}^{ZTR1}) V^{R1F} + (P_{xz\bullet}^{XZR2} \odot P_{yt\bullet}^{YTR2}) V^{R2F} + (P_{yz\bullet}^{YZR3} \odot P_{xt\bullet}^{XTR3}) V^{R3F}
$$

where:
*   $\odot$ is element-wise product.
*   $P$ tensors stack planes (e.g., $P_{xy}^{XYR1}$ stacks all $M_r^{XY}$).
*   $V$ tensors stack vectors (e.g., $V^{R1F}$ stacks all $v_r^1$).
*   Subscripts with $\bullet$ denote bilinear interpolation.
*   An alternative stacked formulation is: $[P_{xy}^{XYR1} \odot P_{xt}^{ZTR1}, P_{xz}^{XZR2} \odot P_{yt}^{YTR2}, P_{yz}^{YZR3} \odot P_{xt}^{XTR3}]^T V^{RF}$ [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize six feature planes ($M^{XY}, M^{XZ}, M^{YZ}, M^{ZT}, M^{YT}, M^{XT}$) and three feature vectors ($v^1, v^2, v^3$) with random values. Initialize a tiny MLP for color regression. Initialize an emptiness voxel.
2.  **Ray Sampling:** For each ray, sample points $(x, y, z, t)$ along the ray. If using spherical coordinates for unbounded scenes, re-parameterize $(x, y, z)$ to $(\theta, \phi, r)$ and sample $r$ linearly between 0 and 1 [S1].
3.  **Feature Query:** For each point, query the HexPlane to get a feature vector $f$ by performing six bilinear interpolations (two for each of the three terms) and one vector-matrix product per term [S1].
4.  **Opacity and Appearance:** Query opacity directly from one HexPlane. Query appearance features from a separate HexPlane only for points with high opacities [S1].
5.  **Color Regression:** Pass the fused feature vector, point coordinates, and view direction to a tiny MLP to regress RGB color [S1].
6.  **Rendering:** Render images via differentiable volumetric rendering along rays using the predicted color and opacity [S1].
7.  **Loss Calculation:** Compute photometric loss between rendered images and ground-truth images. Apply Total Variational (TV) loss on planes for spatial-temporal continuity and depth smooth loss to reduce artifacts [S1].
8.  **Optimization:** Update HexPlane parameters and MLP weights using Adam optimizer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def hexplane_query(x, y, z, t, planes, vectors, R1, R2, R3):
    """
    Query HexPlane for feature vector at (x, y, z, t).
    planes: dict with keys 'XY', 'XZ', 'YZ', 'ZT', 'YT', 'XT'
            each value is a numpy array of shape (dim_A, dim_B, R)
    vectors: list of 3 numpy arrays of shape (F,)
    R1, R2, R3: number of components for each term
    """
    # Term 1: XY and ZT
    # Extract features from XY plane at (x, y) -> shape (R1,)
    feat_xy = bilinear_interpolate(planes['XY'], x, y)
    # Extract features from ZT plane at (z, t) -> shape (R1,)
    feat_zt = bilinear_interpolate(planes['ZT'], z, t)
    # Element-wise product
    term1 = feat_xy * feat_zt
    
    # Term 2: XZ and YT
    feat_xz = bilinear_interpolate(planes['XZ'], x, z)
    feat_yt = bilinear_interpolate(planes['YT'], y, t)
    term2 = feat_xz * feat_yt
    
    # Term 3: YZ and XT
    feat_yz = bilinear_interpolate(planes['YZ'], y, z)
    feat_xt = bilinear_interpolate(planes['XT'], x, t)
    term3 = feat_yz * feat_xt
    
    # Concatenate terms
    fused = np.concatenate([term1, term2, term3])
    
    # Multiply by stacked vectors V_RF
    # V_RF shape: (3*R, F) where R = R1+R2+R3
    # vectors are stacked: [v1, v2, v3]
    V_RF = np.vstack([vectors[0], vectors[1], vectors[2]])
    feature_vector = fused @ V_RF
    
    return feature_vector

def bilinear_interpolate(grid, u, v):
    """
    Perform bilinear interpolation on a 2D grid.
    grid: numpy array of shape (H, W, R)
    u, v: coordinates in [0, 1]
    Returns: numpy array of shape (R,)
    """
    H, W, R = grid.shape
    # Map u, v to grid coordinates
    u_idx = u * (W - 1)
    v_idx = v * (H - 1)
    
    # Get corner indices
    u0 = int(np.floor(u_idx))
    v0 = int(np.floor(v_idx))
    u1 = u0 + 1
    v1 = v0 + 1
    
    # Clamp indices
    u0 = np.clip(u0, 0, W - 2)
    u1 = np.clip(u1, 0, W - 1)
    v0 = np.clip(v0, 0, H - 2)
    v1 = np.clip(v1, 0, H - 1)
    
    # Interpolation weights
    w_u = u_idx - u0
    w_v = v_idx - v0
    
    # Get corner values
    val00 = grid[v0, u0, :]
    val01 = grid[v0, u1, :]
    val10 = grid[v1, u0, :]
    val11 = grid[v1, u1, :]
    
    # Bilinear interpolation
    top = val00 * (1 - w_u) + val01 * w_u
    bottom = val10 * (1 - w_u) + val11 * w_u
    result = top * (1 - w_v) + bottom * w_v
    
    return result

def train_hexplane(images, poses, timestamps, hexplane_params, mlp_params, iterations):
    """
    Train HexPlane model.
    """
    optimizer = Adam([hexplane_params, mlp_params], lr=0.02)
    
    for i in range(iterations):
        # Sample rays
        rays = sample_rays(images, poses, timestamps)
        
        # Render
        colors, opacities = render_rays(rays, hexplane_params, mlp_params)
        
        # Compute loss
        gt_colors = get_gt_colors(rays, images)
        photometric_loss = np.mean((colors - gt_colors) ** 2)
        tv_loss = compute_tv_loss(hexplane_params)
        depth_loss = compute_depth_loss(opacities)
        loss = photometric_loss + lambda_reg * (tv_loss + depth_loss)
        
        # Update
        optimizer.step(loss)
        
        # Coarse to fine: adjust grid resolution if needed
        if i in [70000, 140000, 210000]:
            upsample_grids(hexplane_params)
            
        # Update emptiness voxel
        if i in [50000, 100000]:
            update_emptiness_voxel(hexplane_params)
```

**Parameters:**
*   **Spatial Resolution $N$:** Example value: 512 [S1].
*   **Temporal Resolution $T$:** Example value: 32 [S1].
*   **Feature Size $F$:** Example value: 3 (for RGB colors) [S1].
*   **Basis Number $R$:** Typical values tested: 8, 12, 16, 24, 48, 96. Increasing $R$ leads to better results but more computations; unsuitable large $R$ may lead to overfitting [S1].
*   **Plenoptic Video Dataset:** Appearance HexPlane basis numbers are $R_1=48, R_2=24, R_3=24$ for XY-ZT, XZ-YT, YZ-XT planes respectively. Opacity HexPlane basis numbers are $R_1=24, R_2=12, R_3=12$ [S1].
*   **D-NeRF Dataset:** Appearance HexPlane basis numbers are $R_1=R_2=R_3=48$. Opacity HexPlane basis numbers are $R_1=R_2=R_3=24$ [S1].
*   **Optimization:** Adam optimizer with $\beta_1=0.9, \beta_2=0.99$. Learning rate for feature planes is 0.02, for VRF and neural network is 0.001. All learning rates are exponentially decayed [S1].
*   **Regularization:** Total Variational (TV) loss is applied on all feature planes with $\lambda=0.0005$ for spatial axes and $\lambda=0.001$ for temporal axes [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Bilinear interpolation requires careful handling of boundary conditions [S1].
*   **Boundaries:** HexPlane is limited to bounded scenes because grid sampling fails for out-of-boundary points. NDC requires bounded x, y values and faces forwarding assumptions, leading to artifacts and incorrect geometries at extreme views [S1].
*   **Degenerate Input:** HexPlane generates degraded results when objects move too fast or there are too few observations to synthesize details [S1].
*   **Artifacts:** Explicit representations like HexPlane are prone to artifacts and color jittering, requiring strong regularizations [S1].

## Operating characteristics
*   **Time Complexity:** Querying a HexPlane requires six bilinear interpolations and one vector-matrix product per term [S1].
*   **Memory Complexity:** Memory usage is reduced to $O(N^2 R + N T R + R F)$, where $N$ is spatial resolution, $T$ is time resolution, $R$ is rank, and $F$ is feature dimension [S1].
*   **Measured Performance:** On the Plenoptic Video dataset, HexPlane trained with 650k iterations achieves 29.470 PSNR, 0.018 D-SSIM, 0.078 LPIPS, 8.16 JOD, in 12 GPU hours (112x speedup over DyNeRF) [S1]. On Plenoptic Video dataset, HexPlane trained with 100k iterations (HexPlane†) achieves 29.263 PSNR, 0.020 D-SSIM, 0.097 LPIPS, 8.14 JOD, in 2 GPU hours (672x speedup over DyNeRF) [S1]. On D-NeRF dataset, HexPlane achieves 31.04 PSNR, 0.97 SSIM, 0.04 LPIPS, in 11 minutes training time [S1].
*   **Scaling:** Higher resolutions of the space grid lead to better synthesis quality. HexPlane results are not noticeably affected by a smaller time grid resolution [S1].
*   **Strengths:** Fast training time, compact model size (200MB), no need for deformation fields [S1].
*   **Limitations:** Limited to bounded scenes, prone to artifacts and color jittering, degraded results for extremely sparse observations or fast-moving objects [S1].
*   **When to use:** When fast training and compact representation are prioritized, and scenes are bounded with sufficient observations [S1].
*   **When not to use:** For unbounded scenes without spherical coordinate re-parameterization, or for scenes with extremely sparse observations where deformation fields might be more effective [S1].

## Background
HexPlane was introduced by Ang Cao and Justin Johnson from the University of Michigan, Ann Arbor [S1]. It builds on factored representations for static scenes [5, 7, 55] and TensoRF [7] [S1]. It is most closely related to K*-Planes [14] [S1].

## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
