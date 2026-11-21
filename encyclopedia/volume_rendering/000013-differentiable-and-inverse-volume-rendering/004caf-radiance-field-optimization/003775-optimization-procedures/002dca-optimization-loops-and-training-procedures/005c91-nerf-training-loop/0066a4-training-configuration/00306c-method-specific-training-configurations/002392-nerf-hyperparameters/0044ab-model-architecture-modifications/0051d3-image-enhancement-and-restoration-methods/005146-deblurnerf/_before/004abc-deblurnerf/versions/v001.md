# DeblurNeRF

Also called: DeRF, Deblur-NeRF, Deblurred Neural Radiance Fields

## Overview
DeblurNeRF (also known as DeRF) is a method for constructing deblurred neural radiance fields from blurry input images by modeling the blur kernel as pixel-wise independent ray transformations and composition weights [S2]. It differs from sibling methods like DP-NeRF by using a Deformable Sparse Kernel (DSK) that treats blur as local, pixel-specific deformations rather than scene-wide rigid camera motions, allowing for efficient approximation of spatially varying blur at the cost of 3D geometric consistency [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of blurry images $B$, camera parameters (origin $o$, direction $d$) for each pixel, and a latent scene embedding $l_s$.
*   **Outputs:** A deblurred radiance field function $F_\Theta$ and a blur kernel predictor $G_\Phi$. During inference, the method yields sharp views by bypassing the blur kernel.

**Data Structures:**
*   **NeRF MLP ($F_\Theta$):** Standard NeRF architecture mapping positional encoded $(\mathbf{x}, \mathbf{d})$ to density $\sigma$ and color $\mathbf{c}$ [S2].
*   **Blur Kernel MLP ($G_\Phi$):** An MLP with parameters $\Phi$.
    *   **Inputs:**
        *   $\chi$: Randomly initialized canonical coordinates within a specific small range, representing latent embedded pixel coordinates [S2].
        *   $l_s$: Latent embedded scene information [S2].
    *   **Outputs:** For each transformed ray index $q \in \{1, \dots, n\}$, the MLP outputs a tuple $(\Delta \mathbf{v}_{o;q}, \Delta \mathbf{v}_{T;q}, m_q)$ [S2].
        *   $\Delta \mathbf{v}_{o;q}$: A 3D vector representing the offset to the ray origin [S2].
        *   $\Delta \mathbf{v}_{T;q}$: A 2D vector representing the transformation to the ray direction (applied to pixel coordinates) [S2].
        *   $m_q$: A scalar composition weight [S2].

**Numbered Steps:**
1.  **Ray Generation:** For each pixel $p$, generate a ray $r$ with origin $o$ and direction $d$ based on camera parameters [S2].
2.  **Coarse Sampling:** Divide the ray interval $[t_n, t_f]$ into $N$ intervals using stratified sampling to generate coarse samples $r_i = o + t_i d$ [S2].
    $$
    t_i \sim U\left[t_n + \frac{i-1}{N}(t_f - t_n), t_n + \frac{i}{N}(t_f - t_n)\right]
    $$
3.  **Coarse Rendering:** Render coarse color $\hat{C}_c(r)$ using volume rendering with samples $(\mathbf{c}_i, \sigma_i)$ [S2]:
    $$
    \hat{C}_c(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and $\delta_i = t_{i+1} - t_i$ [S2].
4.  **Hierarchical Sampling:** Compute normalized weights $\hat{w}_i = w_i / \sum_{j=1}^N w_j$ as a PDF and perform hierarchical volume sampling again to produce fine samples [S2].
5.  **Fine Rendering:** Render fine color $\hat{C}_f(r)$ using the same volume rendering process with the fine samples [S2].
6.  **Blur Kernel Prediction:** Predict blur kernel parameters via $G_\Phi(\chi, l_s)$ to obtain $(\Delta \mathbf{v}_{o;q}, \Delta \mathbf{v}_{T;q}, m_q)$ for $q \in \{1, \dots, n\}$ [S2].
7.  **Ray Transformation:** Transform the original ray $r$ into $n$ transformed rays $r_q$ to approximate the blurring process [S2]:
    $$
    r_q = (o + \Delta \mathbf{v}_{o;q}) + t \mathbf{d}_q
    $$
    where $\mathbf{d}_q$ is the transformed ray direction obtained by applying $\Delta \mathbf{v}_{T;q}$ to the pixel coordinates [S2].
8.  **Blurred Color Composition:** Render colors $\hat{C}_q$ for each transformed ray $r_q$ and composite the final blurred color $\hat{B}$ for the pixel [S2]:
    $$
    \hat{B} = \sum_{q \in B(p)} m_q \hat{C}_q
    $$
    where $B(p) = \{1, \dots, n\}$ is the set of indices of the approximated blurring rays [S2].
9.  **Loss Computation:** Train the model using the L2 reconstruction loss between the predicted blurred colors and the ground truth blurry images [S2]:
    $$
    L_{\text{recon}} = \sum_{r \in R} \left( \|\hat{B}_c(r) - B(r)\|_2^2 + \|\hat{B}_f(r) - B(r)\|_2^2 \right)
    $$
    where $R$ is the set of rays in the batch and $B(r)$ is the ground truth RGB color [S2].

**Parameters:**
*   $N$: Number of samples per ray, deciding the approximation quality of the discrete transformation [S2].
*   $n$: Number of transformed rays per pixel, controlling the granularity of the blur kernel [S2].
*   $f$: Frequency band index for positional encoding, $f \in \{0, \dots, m-1\}$, where $m$ is the max frequency [S2].
*   $\chi$: Randomly initialized canonical coordinates within a specific small range [S2].
*   $l_s$: Latent embedded scene information [S2].

**Edge Cases and Pitfalls:**
*   **3D Inconsistency:** The pixel-wise independent optimization of the blur kernel incurs inconsistency in 3D geometry and appearance across different views [S2].
*   **Severe Blur:** While efficient, this compact approximation is suited for moderate blur; severe motion blur may require physically grounded formulations like BAD-NeRF for better pose accuracy and sharper results [S1].

## Operating characteristics
*   **Complexity:** The method adds the computational cost of the MLP $G_\Phi$ and the rendering of $n$ additional rays per pixel. The complexity scales linearly with $n$ and $N$ [S2].
*   **Performance:** DeblurNeRF offers a compact and efficient approximation suited for moderate blur [S1]. It requires only blurry inputs for training and yields sharp views at inference once the kernel is removed [S1].
*   **Limitations:** The lack of 3D consistency in the blur kernel modeling can lead to artifacts in complex scenes or under severe motion blur compared to methods that enforce rigid camera motion constraints (e.g., DP-NeRF) [S1, S2].

## Background
DeblurNeRF (published November 2021) was one of the first methods to address blur degradation (camera motion and defocus) in NeRF by constructing deblurred neural radiance fields (DeRF) from only blurry images [S1, S2]. It was motivated by the need to handle real-world blurry inputs without requiring sharp ground truth data [S2]. It differs from subsequent works like DP-NeRF by prioritizing computational efficiency and local blur modeling over strict physical consistency of camera motion [S1, S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
