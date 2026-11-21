# SfMNeRF
Also called: SfM-NeRF

## Overview
SfMNeRF is a method for novel view synthesis and 3D-scene geometry reconstruction that improves Neural Radiance Fields (NeRF) under sparse input conditions by integrating Structure-from-Motion (SfM) constraints into the training loop [S1]. It explicitly constrains the 3D-scene geometry using matched feature positions, epipolar geometry, and multi-view photometric consistency, thereby reducing geometric ambiguity and preventing the model from getting stuck in local minima [S1]. Unlike standard NeRF which relies solely on implicit volume rendering losses, SfMNeRF adds explicit geometric priors derived from sparse feature matches and cross-view consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images with known camera intrinsics ($K$) and global poses ($[\mathbf{r}_t, \mathbf{t}_t]$). The method operates on pairs of images: a reference image ($I_r$) and a target image ($I_t$) with overlapping regions.
*   **Outputs:** An optimized NeRF model (MLP parameters $\theta$) that predicts volume density $\sigma$ and color $\mathbf{c}$, producing high-quality novel views and accurate depth maps.

**Data Structures:**
*   **Camera Intrinsic Matrix ($K$):** $3 \times 3$ matrix mapping 3D world coordinates to 2D image coordinates.
*   **Relative Pose ($T_{r \to t}$):** Defined as $[\mathbf{r}, \mathbf{t}]$, where $\mathbf{r} = \mathbf{r}_t^{-1}$ and $\mathbf{t} = -\mathbf{r}_t^{-1}\mathbf{t}_t$ [S1].
*   **Mask Map ($M$):** A binary mask where $M(p(k)) = 1$ if point $p(k)$ lies within the minimum bounding rectangle of all matched SIFT features in the reference image, and $0$ otherwise [S1].
*   **SIFT Features:** Sparse 2D keypoints extracted from images, matched across views.
*   **Depth Map ($D$):** Estimated depth for each pixel, derived from the NeRF's volume rendering integration.

**Numbered Steps:**

1.  **Feature Extraction and Matching:**
    Extract Scale-Invariant Feature Transform (SIFT) features from the reference image $I_r$ and target image $I_t$. Obtain matched SIFT correspondences between the two images [S1].

2.  **Sub-pixel Rendering Setup:**
    Randomly select a patch $\Omega_p$ (size $32 \times 32$) from the reference image [S1]. For each pixel $p=(x,y)$ in the patch, generate sub-pixel coordinates by adding random offsets:
    $$p = (x + x_{off}, y + y_{off})$$
    where $x_{off}, y_{off} \in (0, 1)$ [S1].

3.  **Ground Truth Sub-pixel Color Calculation:**
    Compute the ground-truth color $\hat{I}(p)$ for the sub-pixel using bilinear interpolation of the four neighboring integer pixels ($p^{ij}$) from the reference image:
    $$\hat{I}(p) = \sum_{i \in \{top, bottom\}, j \in \{left, right\}} w^{ij} I(p^{ij})$$
    where weights $w^{ij}$ are linearly proportional to the Euclidean distance between $p$ and $p^{ij}$ [S1].

4.  **3D Coordinate Estimation for Features:**
    For each matched SIFT feature $k$, estimate its 3D coordinate $\mathbf{x}_s^k$ by inputting the feature's 2D position and viewing direction into the NeRF. The 3D coordinate is a weighted sum of samples along the ray:
    $$\mathbf{x}_s = \sum_{j=1}^{N_c} w_j (\mathbf{o} + t_j \mathbf{d})$$
    where $w_j = T_j (1 - \exp(-\sigma_j \delta_j))$, $\delta_j = t_{j-1} - t_j$, $N_c$ is the number of samples (typically 64 for coarse/importance sampling combined), $\mathbf{o}$ is the camera origin, and $\mathbf{d}$ is the viewing direction [S1].

5.  **Epipolar Point Filtering:**
    For a point in the reference image, determine the epipolar line in the target image. Filter implausible points on this line by color similarity: keep only points where the color difference between the reference point and the epipolar point is below a threshold. This reduces computational cost and handles occlusions [S1].

6.  **Loss Computation:**
    Compute the following loss terms:

    *   **Rendering Loss ($L_{ren}$):** Mean squared error between the synthesized sub-pixel color $\tilde{I}(p_i)$ and the ground-truth interpolated color $\hat{I}(p_i)$:
        $$L_{ren} = \frac{1}{m} \sum_{p_i \in \Omega_p} \left\| \hat{I}(p_i) - \tilde{I}(p_i) \right\|_2^2$$
        where $m$ is the number of pixels in the patch [S1].

    *   **Positions-of-Matched Features Loss ($L_{3D}$):** Minimizes the distance between the estimated 3D coordinates of matched SIFT features in the reference and target images:
        $$L_{3D} = \frac{1}{n} \sum_{k=1}^n \|\mathbf{x}_s^k - \mathbf{x}_{s'}^k\|_2^2$$
        where $\mathbf{x}_s^k$ and $\mathbf{x}_{s'}^k$ are the 3D coordinates of matched feature $k$ in the reference and target images, respectively [S1].

    *   **Epipolar Constraint Loss ($L_{epi}$):** Minimizes the distance between the estimated 3D coordinate of a reference point $d_{p_r}$ and the closest point on the epipolar line in the target image $d_{p_t}$:
        $$L_{epi} = \min_t \left\{ \left\| d_{p_r} - d_{p_t} \right\|_2^2 \right\}$$
        where $d_{p_r}$ and $d_{p_t}$ are estimated using Eq. (7) [S1].

    *   **Photometric Reconstruction Loss ($L_{pr}$):** Warps pixels from the reference patch to the target view and compares colors. First, project the 3D coordinate $\mathbf{x}_p$ of a pixel $p(k)$ to the target image:
        $$\hat{p}_t = K T_{r \to t} \mathbf{x}_p$$
        Then compute the L1 difference between the reference patch color $\hat{I}_r(p(k))$ and the warped target patch color $\hat{I}_t(\hat{p}_t(k))$, masked by $M$:
        $$L_{pr} = \sum_{p(k) \in \Omega_p} M(p(k)) \left| \hat{I}_r(p(k)) - \hat{I}_t(\hat{p}_t(k)) \right|$$
        [S1].

    *   **SSIM Loss ($L_{SSIM}$):** Structural Similarity index between the reference and warped target patches, masked by $M$:
        $$L_{SSIM} = M \left( \frac{1 - SSIM_{r \to t}}{2} \right)$$
        [S1].

    *   **Depth Smooth Loss ($L_{ds}$):** Enforces smoothness on the estimated depth map $D$, weighted by image gradients to preserve edges:
        $$L_{ds} = \sum_{p(k) \in \Omega_p} \left| \nabla D(p(k)) \cdot \left( e^{-|\hat{I}(p(k))|} \right) \right|$$
        where $\nabla$ is the 2D differential operator [S1].

    *   **Total Loss:**
        $$L_{total} = \lambda_{ren} L_{ren} + \lambda_{3D} L_{3D} + \lambda_{pr} L_{pr} + \lambda_{epi} L_{epi} + \lambda_{SSIM} L_{SSIM} + \lambda_{ds} L_{ds}$$
        [S1].

7.  **Optimization:**
    Update NeRF parameters using the Adam optimizer ($\beta_1=0.9, \beta_2=0.999$, learning rate $1e-3$) to minimize $L_{total}$ [S1].

8.  **Pyramidal Strategy (Optional but Recommended):**
    To prevent local minima, implement training in a pyramid of $L$ levels.
    *   Construct pyramid representations of reference and target images ($I^0_r \dots I^{L-1}_r$ and $I^0_t \dots I^{L-1}_t$), where $I^n$ is half the resolution of $I^{n-1}$ [S1].
    *   Start optimization at the lowest level $I^{L-1}_r, I^{L-1}_t$ to obtain depth map $D^{L-1}_r$ [S1].
    *   Recursively move to level $n-2$. Add a depth loss $L_{depth}$ to use the prior from the previous step:
        $$L_{depth} = \sum_{p(k) \in \Omega_p} |D(p(k)) - D_r^{L-1}(p(k)/2)|$$
        where $D_r^{L-1}(p(k)/2)$ is sampled via bilinear interpolation from the lower-resolution depth map [S1].
    *   The model trained at the "zeroth" level ($I^0$) is the final model [S1].

**Parameters:**
*   Loss weights: $\lambda_{ren} = 1.0$, $\lambda_{3D} = 1.0$, $\lambda_{pr} = 0.001$, $\lambda_{epi} = 0.0001$, $\lambda_{SSIM} = 0.008$, $\lambda_{ds} = 0.01$ [S1].
*   Patch size: $32 \times 32$ [S1].
*   Samples per ray: 64 (coarse + importance) [S1].

**Edge Cases and Pitfalls:**
*   **Epipolar Points:** A large number of epipolar points causes computational inefficiency. Occluded points may not have corresponding points on the epipolar line, requiring color-similarity filtering to remove implausible points [S1].
*   **Local Minima:** Without the pyramidal strategy, the photometric consistency loss may get stuck in local minima due to large pixel motions between views [S1].
*   **Sparse Input:** Standard NeRF fails to obtain moderate 3D structures under sparse input; SfMNeRF is specifically designed to address this [S1].

## Operating characteristics
*   **Time Complexity:** The method adds significant overhead to standard NeRF training due to SIFT extraction, epipolar line sampling, and multi-view warping operations. The epipolar constraint requires searching along epipolar lines, which is computationally expensive if not filtered [S1].
*   **Memory Complexity:** Requires storing SIFT features, matched correspondences, and intermediate depth maps for the pyramidal strategy. The patch-based photometric loss requires storing reference and target image patches [S1].
*   **Performance:** Evaluated on LLFF-NeRF, ScanNet, and DTU MVS datasets. It surpasses state-of-the-art approaches given sparse inputs (2, 5, and 8 input views) [S1].
*   **Strengths:** Improves geometric accuracy and novel view synthesis quality under sparse input conditions by explicitly constraining 3D structure [S1].
*   **Limitations:** Relies on the availability of matched features (SIFT), which may be sparse in textureless regions. Computationally heavier than vanilla NeRF due to explicit geometric constraints [S1].

## Background
SfMNeRF was proposed to address the failure of existing NeRF-based methods to obtain moderate 3D structures under sparse input conditions, where novel view synthesis quality drops dramatically due to implicitly reconstructed inaccurate 3D-scene structures [S1]. The method leverages knowledge from self-supervised depth estimation and Structure-from-Motion (SfM) to constrain geometry during view synthesis training [S1]. It builds on the foundational NeRF framework [1] and incorporates ideas from SaNeRF regarding the use of sparse keypoints for geometry guidance [S1]. The method was evaluated on public datasets including LLFF-NeRF, ScanNet, and DTU MVS [S1].

## Sources
[S1] Improving neural radiance fields with depth-aware optimization for novel view synthesis, https://doi.org/10.1016/j.imavis.2026.106052, fetched 2026-10-26
