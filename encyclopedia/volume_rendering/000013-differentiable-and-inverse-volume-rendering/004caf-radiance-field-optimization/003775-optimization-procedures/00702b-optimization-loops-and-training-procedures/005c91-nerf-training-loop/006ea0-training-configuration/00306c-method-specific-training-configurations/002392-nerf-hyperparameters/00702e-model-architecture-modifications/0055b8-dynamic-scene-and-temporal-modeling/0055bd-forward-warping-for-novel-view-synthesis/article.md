# Forward Warping for Novel View Synthesis

Also called: Canonical Field Forward Warping, Forward Flow

## Overview
Forward Warping for Novel View Synthesis is a method for rendering dynamic scenes by modeling a static scene at a canonical time step and forward warping the entire canonical field to other time steps [S1]. It addresses the many-to-one and one-to-many mapping difficulties inherent in dynamic scene reconstruction using differentiable forward warping with average splatting and an inpaint network [S1]. Unlike backward flow methods such as NDVG, this approach leverages a canonical representation to simplify the deformation field, achieving state-of-the-art performance on public datasets, particularly for canonical frames [S1].

## How it works
The method employs a coarse-to-fine training strategy. The coarse stage learns a proxy geometry to define the bounding box for the fine stage, while the fine stage refines the radiance field and deformation [S1]. The canonical time is set to the first frame for the D-NeRF dataset and the middle frame for the HyperNeRF dataset [S1].

### Inputs and Outputs
*   **Inputs:** A sequence of images $\{I_t\}$ capturing a dynamic scene, with known camera poses and timestamps $t$.
*   **Outputs:** A rendered image $\mathbf{C}(\mathbf{r})$ for a query ray $\mathbf{r}$ at a target time $t$, and the underlying voxel grids representing the scene.

### Data Structures
The method relies on several voxel grids and neural networks. All voxel grids are 3D structures indexed by spatial coordinates.

1.  **Canonical Radiance Feature Grid ($VR_f$):** Voxel grid containing radiance features. Dimension of features is 12 [S1].
2.  **Canonical Coordinate Grid ($V_{rCan}$):** Voxel grid containing the coordinate of each voxel [S1].
3.  **Canonical Density Grid ($V_{\sigma}^{Can}$):** Voxel grid containing canonical density [S1].
4.  **Canonical Color Feature Grid ($V_{cf}^{Can}$):** Voxel grid containing canonical color features. Dimension is 3 or 12 depending on the training stage [S1].
5.  **Canonical Deformation Feature Grid ($VD_f^{Can}$):** Voxel grid containing canonical deformation features. Dimension is 0 for D-NeRF, 12 for HyperNeRF [S1].
6.  **Trajectory Grid ($VT_t$):** Voxel grid containing DCT parameters of each voxel's trajectory [S1].
7.  **Deformation Flow Grid ($V_{flow}$):** Voxel grid containing deformation flow of each voxel from canonical to time $t$ [S1].
8.  **Warped Radiance Field ($VR_t$):** Radiance field at time $t$ warped from $VR_{Can}$ [S1].
9.  **Inpainted Radiance Field ($VR_t^{Inp}$):** Radiance field at time $t$ after inpainting by $F_{\theta3}$ [S1].
10. **Upsampled Radiance Field ($VR_t^{Up}$):** Upsampled radiance field at time $t$ [S1].

### Neural Networks
1.  **Canonical Radiance Network ($F_{\theta1}$):**
    *   **Structure:** 3-layer MLP, width 128, ReLU activation [S1].
    *   **Input:** Radiance feature sampled from $VR_f$ (dim 12) and embedded position (dim 33) [S1].
    *   **Output:** Density (dim 1) and color feature (dim 3 or 12) [S1].
2.  **Canonical Deformation Network ($F_{\theta2}$):**
    *   **Structure:** 4-layer MLP, width 64 [S1].
    *   **Input:** Deformation feature and embedded position (dim 33) [S1].
    *   **Deformation Feature Dimension:** 0 for D-NeRF, 12 for HyperNeRF [S1].
    *   **DCT Bases:** 15 for D-NeRF, 25 for HyperNeRF [S1].
3.  **View Dependent Color Network ($F_{\theta4}$):**
    *   **Structure:** 2-layer MLP, width 64 [S1].
    *   **Input:** Embedded view direction (dim 27) and color feature (dim 12) [S1].
    *   **Output:** RGB color (dim 3) [S1].
4.  **Inpaint Network ($F_{\theta3}$):**
    *   **Structure:** UNet with 4 encode layers and 3 decode layers, plus an upsample layer [S1].
    *   **Encode Layer:** One max pooling layer and two convolution layers [S1].
    *   **Decode Layer:** One up-sample interpolation layer and two convolution layers [S1].
    *   **Convolution Layer:** Instance norm followed by convolution and ReLU [S1].

### Algorithm Steps

1.  **Coarse Stage Initialization:**
    *   Set expected voxel number to $6^3$ [S1].
    *   Set inpaint network scale factor to 1.5 [S1].
    *   Train for 20,000 iterations [S1].
    *   Do not use the inpaint network during this stage [S1].
    *   Purpose: Learn proxy geometry to calculate the bounding box for the fine stage [S1].

2.  **Fine Stage Initialization:**
    *   Set expected voxel number to $8^3$ for D-NeRF, $7^3$ for HyperNeRF [S1].
    *   Set inpaint network scale factor to 2.0 [S1].
    *   Train for 100,000 iterations [S1].
    *   Initialize with 10 images with the closest time steps to the canonical step [S1].
    *   Progressively add one image with the closest time step every 60 iterations [S1].

3.  **Forward Warping Pipeline (for a target time $t$):**
    *   **Deformation Estimation:** For each voxel in the canonical grid, use $F_{\theta2}$ to estimate the trajectory using DCT bases. Compute the deformation flow $V_{flow}$ from canonical position to time $t$ [S1].
    *   **Average Splatting:** Warp the canonical radiance features from $VR_{Can}$ to time $t$ using average splatting to produce $VR_t$ [S1]. Average splatting is used instead of softmax splatting to avoid complexity and because softmax weights showed no obvious improvement [S1].
    *   **Inpainting:** Pass $VR_t$ through the Inpaint Network $F_{\theta3}$ to produce the inpainted radiance field $VR_t^{Inp}$ [S1].
    *   **Upsampling:** Upsample $VR_t^{Inp}$ to produce $VR_t^{Up}$ [S1].

4.  **Rendering:**
    *   Sample points along rays in the warped fields $VR_t^{Inp}$ and $VR_t^{Up}$ [S1].
    *   Use $F_{\theta1}$ to get density and color features from the canonical radiance features [S1].
    *   Use $F_{\theta4}$ to get RGB color from the color features and embedded view direction [S1].
    *   Render colors $\mathbf{C}_{Inp}(\mathbf{r})$ and $\mathbf{C}_{Up}(\mathbf{r})$ using volume rendering [S1].

### Loss Functions
The total loss is a weighted sum of several components:
$$ \mathcal{L} = \mathcal{L}_{photo} + w_1\mathcal{L}_{ptc} + w_2\mathcal{L}_{bg} + w_3\mathcal{L}_{flow} + w_4\mathcal{L}_{vdiff} + w_5\mathcal{L}_{tv}(\mathbf{V}_{\sigma}^{Can}) + w_6\mathcal{L}_{tv}(\mathbf{V}_{flow}^t) + w_7\mathcal{L}_{tv}(\mathbf{D}) $$
where $w_1..w_7$ are balancing weights [S1].

1.  **Point-wise Color Loss ($\mathcal{L}_{ptc}$):** Supervises the color of sampled points weighted by their accumulation [S1].
    $$ \mathcal{L}_{ptc} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \mathcal{L}(\mathbf{C}_{\text{Inp}}(\mathbf{r})) + \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \mathcal{L}(\mathbf{C}_{\text{Up}}(\mathbf{r})) $$
    where
    $$ \mathcal{L}(\mathbf{C}(\mathbf{r})) = \frac{1}{K} \sum_{k=1}^{K} A_{\text{accum}}(k) \|\mathbf{c}(w_k) - \mathbf{C}_{\text{gt}}(\mathbf{r})\|_2^2 $$
    and
    $$ A_{\text{accum}}(k) = T(w_k) \, \sigma(w_k) \delta_k $$
    Here, $\mathbf{C}_{\text{Inp}}$ and $\mathbf{C}_{\text{Up}}$ are colors from inpainted and upsampled fields, $K$ is the number of samples per ray, $T(w_k)$ is the transmittance, $\sigma(w_k)$ is the density, and $\delta_k$ is the distance between samples [S1].

2.  **Background Entropy Loss ($\mathcal{L}_{bg}$):** Encourages densities to concentrate on foreground or background [S1].
    $$ \mathcal{L}_{bg} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \mathcal{L}(\mathbf{A}_{\text{Inp}}(\mathbf{r})) + \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \mathcal{L}(\mathbf{A}_{\text{Up}}(\mathbf{r})) $$
    where
    $$ \mathcal{L}(\mathbf{A}(\mathbf{r})) = -\frac{1}{K-1} \sum_{k=1}^{K-1} [ A_{\text{accum}}(k)\log(A_{\text{accum}}(k)) + (1-A_{\text{accum}}(k))\log(1-A_{\text{accum}}(k)) ] $$
    [S1].

3.  **Total Variation Losses ($\mathcal{L}_{tv}$):** Applied to canonical density grid $\mathbf{V}_{\sigma}^{Can}$, deformation flow grid $\mathbf{V}_{flow}^t$, and deformation feature grid $\mathbf{D}$ to improve visual quality [S1].

### Parameters
*   **Coarse Stage Weights:** $w_1=1e-1, w_2=1e-2, w_3=1e-5, w_4=0., w_5=1e-6, w_6=1e-3, w_7=1e-1$ [S1].
*   **Fine Stage Weights:** $w_1=1e-2, w_2=1e-3, w_3=1e-5, w_4=1e-5, w_5=1e-6, w_6=1e-3, w_7=1e-1$ [S1].
*   **Learning Rate:** $1e-1$ for voxel grids, $1e-3$ for networks [S1].
*   **Optimizer:** Adam [S1].

### Edge Cases and Pitfalls
*   **Memory Consumption:** The method is relatively memory-consuming, especially for real scenes [S1].
*   **Training Speed:** Training is slow, taking approximately one day per scene on a GeForce RTX 3090 [S1].
*   **Coarse Stage Importance:** Removing the coarse stage causes significant performance drops because the voxel grid covers a larger space without proxy geometry filtering [S1].
*   **Inpaint Network Importance:** Removing the inpaint network significantly reduces performance (e.g., Hell Warrior PSNR drops from 27.71 to 27.15) [S1].
*   **Canonical Time Selection:** Setting the canonical time to the middle frame improves performance for HyperNeRF as the geometry is closer to other time steps, but the first frame is used for D-NeRF to compare with physical canonical-based methods [S1].

## Operating characteristics
*   **Time Complexity:** Training takes approximately one day per scene on a GeForce RTX 3090 [S1].
*   **Memory Complexity:** High memory consumption, particularly for real scenes [S1].
*   **Model Size:** 260M parameters for D-NeRF, 440M for HyperNeRF [S1].
*   **Rendering Speed:** 7s per image for D-NeRF, 15s per image for HyperNeRF [S1].
*   **Strengths:** Achieves state-of-the-art performance on public datasets, especially for canonical frames. Forward warping design provides a clear advantage over backward flow methods like NDVG [S1].
*   **Limitations:** High memory usage and slow training speed [S1].
*   **When to Use:** When high-quality novel view synthesis of dynamic scenes is required, and computational resources allow for long training times.
*   **When Not to Use:** When real-time rendering or low-latency training is required, or when memory is constrained.

## Background
Forward Warping for Novel View Synthesis was proposed in the paper "Forward Flow for Novel View Synthesis of Dynamic Scenes" [S1]. The method builds on the concept of canonical representations for dynamic scenes, addressing the challenges of many-to-one and one-to-many mappings through differentiable forward warping. It was developed to improve upon existing methods like NDVG by leveraging a canonical field and forward warping pipeline [S1].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
