# VolSplat

Also called: Differentiable Multi-scale Voxel Module, Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, Voxel-aligned Feed-forward 3DGS

## Overview
VolSplat is a multi-view feed-forward framework for 3D Gaussian Splatting (3DGS) that replaces pixel-aligned Gaussian prediction with voxel-aligned prediction [S1][S3, unofficial]. It directly regresses 3D Gaussian primitives from a predicted 3D voxel grid, overcoming the alignment errors and view-biased density distributions inherent in pixel-aligned methods [S1]. By aggregating multi-view features into a shared 3D voxel container, it decouples feature fusion from camera view frustums, enabling adaptive density control based on scene complexity and robust multi-view consistency [S1][S3, unofficial].

## How it works
VolSplat operates in two main phases: 3D feature construction via voxelization and Gaussian parameter regression from refined voxel features [S1][S3, unofficial]. A variant, $\text{VG}^2$GT, introduces a coarse-to-fine multi-scale voxel strategy [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   $N$ input images $I = \{I_i\}_{i=1}^N$, where $I_i \in \mathbb{R}^{H \times W \times 3}$ [S1].
*   Corresponding camera poses $P = \{P_i\}_{i=1}^N$ (for VolSplat) or unposed images (for $\text{VG}^2$GT) [S1][S2, unofficial].

**Outputs:**
*   A set of 3D Gaussians $G = \{(\mu_g, \Sigma_g, \alpha_g, c_g)\}_{g=1}^M$, where $M$ is the number of occupied voxels [S1].
*   In $\text{VG}^2$GT, additionally predicted camera parameters $\{C_i\}_{i=1}^N$ [S2, unofficial].

### Data Structures
1.  **Image Patch Features ($F_{P_i}$):** Extracted from a Transformer-based backbone for each input image $I_i$. Shape: $\mathbb{R}^{H_p \times W_p \times C}$, where $H_p, W_p$ are patch dimensions [S1][S2, unofficial].
2.  **Global Point Cloud ($\Omega_m$):** A set of 3D points obtained by unprojecting pixels using estimated depth maps and camera parameters [S1][S2, unofficial].
3.  **Voxel Grid:** A sparse 3D grid containing features.
    *   **Coarse Voxel Features ($f_m^v$):** Computed for coarse voxels.
    *   **Fine Voxel Features ($f_m^{dv}$):** Computed for fine voxels (in $\text{VG}^2$GT) [S2, unofficial].
4.  **Gaussian Parameters:**
    *   Center $\mu_g \in \mathbb{R}^3$
    *   Scale $s_g \in \mathbb{R}^3$
    *   Rotation $r_g \in \mathbb{R}^4$ (Quaternion)
    *   Opacity $\sigma_g \in \mathbb{R}$
    *   Color $c_g$ (Spherical Harmonics coefficients) [S1][S2, unofficial].

### Algorithm Steps

#### 1. 2D Feature Extraction and Depth Estimation
1.  Extract 2D image features for each input image $I_i$ using a Transformer-based network (e.g., ViT backbone) to obtain patch features $F_{P_i}$ [S1][S2, unofficial].
2.  Construct per-view cost volumes using plane sweeping [S1][S3, unofficial].
3.  Estimate a depth map $D_i$ for each view using a Depth Prediction Module [S1][S3, unofficial].
    *   *Note:* In $\text{VG}^2$GT, camera parameters $C_i$ are also predicted by a head [S2, unofficial].

#### 2. 3D Feature Construction (Voxelization)
1.  **Unprojection:** Unproject the 2D features into 3D space using the estimated depth maps $D_i$ and camera poses $P_i$ to form a dense global point cloud $\Omega_m$ [S1][S3, unofficial].
    *   For each pixel $p$ in image $i$, compute 3D point $x_p = P_i^{-1}(p, D_i(p))$ [S1][S2, unofficial].
2.  **Voxel Feature Aggregation:** For each voxel $m$ in the grid, compute the voxel feature $f_m^v$ as a confidence-weighted average of features from all 3D points $\Omega_m$ falling inside the voxel [S2, unofficial]:
    $$ f_m^v = \frac{\sum_{p \in \Omega_m} n(p) f(p)}{\sum_{p \in \Omega_m} n(p)} $$
    where $n(p)$ is the depth confidence of point $p$, defined as the minimum point confidence within the voxel: $n_m = \min_{p \in \Omega_m} n(p)$ [S2, unofficial].
    *   *Note:* VolSplat [S1] aggregates features directly into the voxel grid without explicit point-cloud downsampling in the description, while $\text{VG}^2$GT [S2, unofficial] explicitly mentions point-cloud downsampling and interpolation.

#### 3. Feature Refinement
**VolSplat Path:**
1.  Refine the 3D voxel features using a sparse 3D decoder (3D U-Net) [S1][S3, unofficial].
2.  The 3D U-Net predicts residual features $\Delta f_m$ for each voxel [S1].
3.  Add the predicted residual features to the original voxel features: $f_m^{refined} = f_m^v + \Delta f_m$ [S1].

**$\text{VG}^2$GT Path (Coarse-to-Fine):**
1.  **Coarse Enhancement:** Serialize the sparse voxel grid into a 1D feature sequence with operator $S$. Enhance the sequence with PointTransformerV3 ($\gamma_\psi$). Deserialize back to voxel space with $S^{-1}$ [S2, unofficial]:
    $$ \hat{f}_m^v = S^{-1} \circ \gamma_\psi \circ S (f_m^v) $$
2.  **Feature Mapping:** Map enhanced voxel features back to 3D points in $\Omega_m$. For each point $p$, find its 16 nearest voxel centers with KNN. Interpolate enhanced voxel features back to the point by distance-weighted aggregation [S2, unofficial].
3.  **Fusion:** Fuse these enhanced features into backbone patch features using zero convolution, then decode depth maps and camera parameters again [S2, unofficial].
4.  **Fine-Level Construction:** Repeat the aggregation in Step 2 to obtain fine voxel features $f_m^{dv}$ from enhanced image patches. Use a DPT head to upsample image features and align them with the global point cloud, without downsampling the global point cloud [S2, unofficial].
5.  **Fine Enhancement:** Enhance fine voxel features using PointMLP ($\mathcal{M}_\phi$) instead of PTv3 for memory efficiency [S2, unofficial]:
    $$ \hat{f}_m^{dv} = S^{-1} \circ \mathcal{M}_\phi \circ S (f_m^{dv}) $$

#### 4. Gaussian Parameter Regression
For each occupied voxel $m$ with center $x_m$ and refined feature $f_m^{refined}$ (or $\hat{f}_m^{dv}$):

1.  **Initialization:**
    *   Initial Gaussian center $\mu_{g,0} = x_m$ (voxel center) [S2, unofficial].
    *   Initial color $c_{g,0}$ is a weighted average of image colors [S2, unofficial].
    *   Spherical Harmonic (SH) coefficients above order 0 are initialized to 0 [S2, unofficial].
2.  **Residual Prediction:**
    *   Predict residual offsets for Gaussian center $\Delta \mu_g$ and color $\Delta c_g$ using an MLP [S2, unofficial].
    *   Final center: $\mu_g = \mu_{g,0} + \Delta \mu_g$ [S2, unofficial].
3.  **Scale Prediction:**
    *   Predict scale $s_g$ by applying a sigmoid activation with an upper bound of twice the voxel size [S2, unofficial].
4.  **Rotation and Opacity:**
    *   Regress rotation quaternion $r_g$ and opacity $\sigma_g$ directly by MLPs [S2, unofficial].
5.  **Self-Splitting (Optional, $\text{VG}^2$GT):**
    *   Each voxel feature regresses multiple sets of Gaussian primitive parameters to improve fine-grained accuracy [S2, unofficial].

#### 5. Rendering
Render novel views from the predicted 3D Gaussians using differentiable volume rendering [S1][S3, unofficial].
*   In $\text{VG}^2$GT, Continuous Stochastic Solids Volume Rendering is used, where the color at pixel $pix$ is determined by the maximum Gaussian response along the ray $t^*$ [S2, unofficial]:
    $$ I_{pix} = c_G(t^*) $$

### Parameters
*   **Input Images:** $N$ views, resolution $H \times W$ [S1].
*   **Voxel Grid Resolution:** Not given in the sources.
*   **KNN Neighbors:** 16 (in $\text{VG}^2$GT) [S2, unofficial].
*   **Scale Upper Bound:** $2 \times$ voxel size [S2, unofficial].

### Edge Cases and Pitfalls
*   **Alignment Errors:** Pixel-aligned methods suffer from alignment errors when source views contain occlusions or low texture, leading to floaters and view-dependent inconsistency [S1]. VolSplat mitigates this by volumetric aggregation [S1].
*   **Density Control:** Pixel-aligned methods enforce a uniform density distribution, causing redundant primitives in simple regions and failure to capture fine-grained structures [S1]. VolSplat enables adaptive density control based on 3D scene complexity [S1].
*   **Memory Usage:** PointTransformerV3 has substantially higher memory consumption when processing million-point-scale point clouds, potentially leading to GPU out-of-memory errors. PointMLP is used in the fine level to mitigate this [S2, unofficial].

## Subtopics

### [VS-Splat](8542-vs-splat.md)
VS-Splat is a feed-forward framework for 3D object reconstruction from sparse-view 2D images that employs a coarse-to-fine (C2F) strategy to predict 3D Gaussian primitives. It distinguishes itself from sibling methods like LaRa and GenDen by performing voxel-level selection *before* generating fine primitives, rather than selecting existing primitives for refinement. This ensures that the coarse representation determines the spatial location for new fine primitives, enabling adaptive density control without auxiliary supervision.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources, though PointMLP is noted as more memory efficient than PointTransformerV3 for large-scale point clouds [S2, unofficial].
*   **Measured Performance:** Evaluated on RealEstate10K, ScanNet, and ACID benchmarks, achieving state-of-the-art performance [S1].
*   **Strengths:** Robust multi-view consistency, cleaner occlusion handling, adaptive density control, reduced floaters [S1].
*   **Limitations:** Voxel sparsity limits recoverable scene detail [S2, unofficial].
*   **When to use:** When multi-view alignment is challenging due to occlusions or low texture, and adaptive density control is required [S1].
*   **When not to use:** When high-frequency details are critical and voxel sparsity is a bottleneck, or when computational resources are extremely limited for large-scale point cloud processing [S2, unofficial].

## Background
VolSplat was introduced to rethink the widely adopted pixel-aligned Gaussian prediction paradigm in feed-forward 3DGS [S1]. It addresses the limitations of 2D feature matching and rigid pixel-to-Gaussian association [S1]. The method is evaluated on large-scale benchmarks such as RealEstate10K, ScanNet, and ACID [S1]. A related method, $\text{VG}^2$GT, extends this concept with a multi-scale voxel module and joint camera parameter prediction [S2, unofficial].

## Sources
[S1] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
[S2] $\text{VG}^2$GT: Voxel-Gaussian Splatting Visual Geometry Grounded Transformer, https://doi.org/10.48550/arxiv.2606.01573, fetched 2026-10-26 (unofficial)
[S3] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
