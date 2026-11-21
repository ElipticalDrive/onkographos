# SfMNeRF

Also called: SfM-NeRF

## Overview
SfMNeRF is a method for novel view synthesis and 3D-scene geometry reconstruction that improves Neural Radiance Fields (NeRF) under sparse input conditions by integrating Structure-from-Motion (SfM) constraints and depth prediction into the training loop [S1]. It explicitly constrains the 3D-scene geometry using matched feature positions, epipolar geometry, and multi-view photometric consistency, thereby reducing geometric ambiguity and preventing the model from getting stuck in local minima [S1]. Unlike standard NeRF which relies solely on implicit volume rendering, SfMNeRF adds auxiliary geometric loss terms to enforce that identical features share world coordinates and that corresponding points lie on epipolar lines [S1].

## How it works
SfMNeRF optimizes a continuous 5D radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ that outputs volume density $\sigma$ and color $\mathbf{c}$ [S1]. The optimization minimizes a total loss composed of rendering, geometric, and consistency terms. The method employs a pyramidal training strategy to handle large pixel motions and improve convergence [S1].

### Inputs and Outputs
**Inputs:**
*   A set of $m$ training images $\{I_i\}_{i=1}^m$ with known camera intrinsics $K$ and global poses $[\mathbf{r}_i, \mathbf{t}_i]$ [S1].
*   Matched Scale-Invariant Feature Transform (SIFT) correspondences between a reference image $I_r$ and a target image $I_t$ [S1].
*   A patch size $P \times P$ (typically $32 \times 32$) [S1].
*   Pyramid levels $L$ for the pyramidal strategy [S1].

**Outputs:**
*   Optimized NeRF parameters $\theta$ [S1].
*   Synthesized novel view images and depth maps [S1].

### Data Structures
*   **Camera Intrinsic Matrix $K$**: $3 \times 3$ matrix containing focal lengths and principal point [S1].
*   **Global Pose $[\mathbf{r}_i, \mathbf{t}_i]$**: Rotation matrix $\mathbf{r}_i$ and translation vector $\mathbf{t}_i$ for image $i$ [S1].
*   **Relative Pose $T_{r \to t}$**: $4 \times 4$ transformation matrix from reference to target view, defined as $T_{r \to t} = [\mathbf{r}, \mathbf{t}]$ where $\mathbf{r} = \mathbf{r}_t^{-1}$ and $\mathbf{t} = -\mathbf{r}_t^{-1}\mathbf{t}_t$ [S1].
*   **SIFT Matches**: A list of 2D pixel coordinates $\{(p_r^k, p_t^k)\}_{k=1}^n$ for $n$ matched features [S1].
*   **Mask Map $M$**: Binary mask where $M(p(k)) = 1$ if point $p(k)$ is inside the minimum bounding rectangle of all SIFT matches in the reference image, else $0$ [S1].
*   **Depth Map $D$**: Estimated depth for each pixel, derived from the NeRF's density samples [S1].

### Numbered Steps

1.  **Pyramid Construction**: Construct $L$-level pyramid representations for the reference image $I_r$ and target image $I_t$. Let $I_r^0, \dots, I_r^{L-1}$ and $I_t^0, \dots, I_t^{L-1}$ be the pyramid levels, where $I_r^n$ is downsampled from $I_r^{n-1}$ with half the resolution [S1].
2.  **Initialization**: Start optimization at the lowest resolution level $n = L-1$ using images $I_r^{L-1}$ and $I_t^{L-1}$ [S1].
3.  **Sub-pixel Sampling**: For a random patch $\Omega_p$ in the reference image, generate sub-pixel coordinates $p = (x + x_{off}, y + y_{off})$ where $x_{off}, y_{off} \in (0, 1)$ [S1].
4.  **Ground Truth Interpolation**: Compute the ground-truth color $\hat{I}(p)$ for each sub-pixel using bilinear interpolation of the four nearest integer pixels in the reference image [S1]:
    $$
    \hat{I}(p) = \sum_{i \in \{top, bottom\}, j \in \{left, right\}} w^{ij} I(p^{ij})
    $$
    where $w^{ij}$ is proportional to the inverse Euclidean distance between $p$ and neighbor $p^{ij}$ [S1].
5.  **3D Coordinate Estimation**: For each sample $j$ along the ray through $p$, query the NeRF for density $\sigma_j$ and depth $t_j$. Compute the weighted 3D coordinate $\mathbf{x}_s$ of the feature/point:
    $$
    \mathbf{x}_s = \sum_{j=1}^{N_c} w_j (\mathbf{o} + t_j \mathbf{d}), \quad w_j = T_j (1 - \exp(-\sigma_j \delta_j))
    $$
    where $\delta_j = t_{j-1} - t_j$, $T_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \delta_k)$, and $N_c$ is the number of samples (typically 64 for coarse and 64 for fine sampling) [S1].
6.  **Loss Computation**: Calculate the following loss terms:
    *   **Rendering Loss ($L_{ren}$)**: Mean squared error between synthesized color $\tilde{I}(p)$ and ground-truth sub-pixel color $\hat{I}(p)$ [S1]:
        $$
        L_{ren} = \frac{1}{|\Omega_p|} \sum_{p \in \Omega_p} \left\| \hat{I}(p) - \tilde{I}(p) \right\|_2^2
        $$
    *   **Positions-of-Matched Features Loss ($L_{3D}$)**: Enforces that matched SIFT features have identical 3D coordinates in reference and target views [S1]:
        $$
        L_{3D} = \frac{1}{n} \sum_{k=1}^n \|\mathbf{x}_s^k - \mathbf{x}_{s'}^k\|_2^2
        $$
        where $\mathbf{x}_s^k$ and $\mathbf{x}_{s'}^k$ are the 3D coordinates of matched feature $k$ in the reference and target images, respectively [S1].
    *   **Epipolar Constraint Loss ($L_{epi}$)**: Minimizes the distance between the 3D coordinate of a point in the reference image and the epipolar line in the target image [S1]. First, filter implausible points on the epipolar line by color similarity (discard points where color difference exceeds a threshold) [S1]. Then:
        $$
        L_{epi} = \min_t \left\{ \left\| \mathbf{x}_{p_r} - \mathbf{x}_{p_t} \right\|_2^2 \right\}
        $$
        where $\mathbf{x}_{p_r}$ is the 3D coordinate of the reference point and $\mathbf{x}_{p_t}$ is the 3D coordinate of a point on the epipolar line in the target view [S1].
    *   **Photometric Consistency Loss ($L_{pc}$)**: Combines photometric reconstruction loss $L_{pr}$ and SSIM loss $L_{SSIM}$ [S1].
        *   Warp pixels from the reference patch to the target view: $\hat{p}_t = K T_{r \to t} \mathbf{x}_p$ [S1].
        *   Interpolate color $\hat{I}_t(\hat{p}_t)$ in the target image [S1].
        *   $L_{pr}$:
            $$
            L_{pr} = \sum_{p(k) \in \Omega_p} M(p(k)) \left| \hat{I}_r(p(k)) - \hat{I}_t(\hat{p}_t(k)) \right|
            $$
        *   $L_{SSIM}$:
            $$
            L_{SSIM} = M \left( \frac{1 - SSIM_{r \to t}}{2} \right)
            $$
            where $SSIM_{r \to t}$ is the structural similarity index between the reference patch and the warped target patch [S1].
    *   **Depth Smooth Loss ($L_{ds}$)**: Encourages smooth depth variations, weighted by image gradients to preserve edges [S1]:
        $$
        L_{ds} = \sum_{p(k) \in \Omega_p} \left| \nabla D(p(k)) \cdot e^{-|\hat{I}(p(k))|} \right|
        $$
        where $\nabla$ is the 2D differential operator [S1].
    *   **Depth Prior Loss ($L_{depth}$)**: Used in higher pyramid levels to constrain depth using the solution from the lower level [S1]:
        $$
        L_{depth} = \sum_{p(k) \in \Omega_p} |D(p(k)) - D_r^{n-1}(p(k)/2)|
        $$
        where $D_r^{n-1}$ is the depth map from the previous (lower resolution) level, sampled via bilinear interpolation at coordinates $p(k)/2$ [S1].
7.  **Total Loss and Optimization**: Compute the total loss:
    $$
    L_{total} = \lambda_{ren} L_{ren} + \lambda_{3D} L_{3D} + \lambda_{pr} L_{pr} + \lambda_{SSIM} L_{SSIM} + \lambda_{epi} L_{epi} + \lambda_{ds} L_{ds} + \lambda_{depth} L_{depth}
    $$
    Optimize $\theta$ using Adam optimizer ($\beta_1=0.9, \beta_2=0.999$, learning rate $1e-3$) [S1].
8.  **Pyramidal Progression**: After convergence at level $n$, move to level $n-1$. Use the trained model from level $n$ as initialization for level $n-1$. Include $L_{depth}$ in the loss function to propagate the geometric prior from the lower resolution [S1]. Repeat until level $0$ (original resolution) is reached [S1].

### Parameters
*   $\lambda_{ren} = 1.0$
*   $\lambda_{3D} = 1.0$
*   $\lambda_{pr} = 0.001$
*   $\lambda_{epi} = 0.0001$
*   $\lambda_{SSIM} = 0.008$
*   $\lambda_{ds} = 0.01$
*   Patch size: $32 \times 32$ [S1]
*   Samples per ray: 64 (coarse) + 64 (fine) [S1]

### Edge Cases and Pitfalls
*   **Epipolar Point Filtering**: The number of points on an epipolar line is large. Implausible points (occluded or mismatched) must be filtered using color similarity to avoid computational inefficiency and incorrect gradients [S1].
*   **Occlusion**: Corresponding points may not exist on the epipolar line due to occlusion. The mask $M$ and color filtering help mitigate this, but the method assumes sufficient overlap and feature matches [S1].
*   **Local Minima**: Without the pyramidal strategy, the photometric consistency loss may cause the optimizer to get stuck in local minima, especially with large pixel motions [S1].

## Operating characteristics
*   **Time Complexity**: The method is computationally intensive due to the additional geometric constraints (SIFT matching, epipolar geometry calculations, warping) and the pyramidal training strategy which requires multiple passes over the data at different resolutions [S1].
*   **Memory Complexity**: Requires storing pyramid levels of images and depth maps, as well as SIFT features and masks. The use of NVIDIA RTX 3090 suggests significant VRAM usage, particularly for high-resolution inputs (e.g., 504x378 downsampled from 4032x3024) [S1].
*   **Performance**: Evaluated on LLFF-NeRF, ScanNet, and DTU MVS datasets. SfMNeRF surpasses state-of-the-art approaches under sparse input conditions (2, 5, and 8 input views) [S1].
*   **Strengths**: Improves geometric accuracy and novel view synthesis quality in sparse settings by explicitly constraining 3D structure [S1].
*   **Limitations**: Relies on the availability of good SIFT matches and camera poses. Performance may degrade if features are sparse or if the scene has large untextured regions [S1].

## Background
SfMNeRF was proposed to address the failure of existing NeRF-based methods to obtain moderate 3D structures under sparse input conditions, where novel view synthesis quality drops dramatically due to implicitly reconstructed inaccurate 3D-scene structures [S1]. The method leverages knowledge from self-supervised depth estimation and Structure-from-Motion (SfM) to constrain the geometry during view synthesis training [S1]. It builds upon the standard NeRF framework [1] and incorporates ideas from SaNeRF regarding the use of sparse keypoints [S1]. The method was evaluated on LLFF-NeRF, ScanNet, and DTU MVS datasets [S1].

## Sources
[S1] Improving neural radiance fields with depth-aware optimization for novel view synthesis, https://doi.org/10.1016/j.imavis.2026.106052, fetched 2026-10-26
