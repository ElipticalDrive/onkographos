# C-NeRF


## Overview
C-NeRF is a voxel-based radiance field representation that uses a 3D convolutional neural network (CNN) to generate feature volumes, which are then decoded by a NeRF-like multilayer perceptron (MLP) to predict density and color [S1]. It distinguishes itself from its parent Voxel-based Representation by explicitly decoupling the spatial feature extraction (via 3D CNN) from the view-dependent appearance modeling (via MLP), addressing the limitations of pure MLP approaches in capturing high-frequency details [S1].

## How it works
**Inputs:** A set of calibrated images with known camera poses.
**Outputs:** A continuous radiance field function $(\sigma, \mathbf{c})$ defined over 3D space and viewing direction.

**Data Structures:**
1.  **3D Feature Volume:** A dense 3D grid of fixed resolution. Each voxel stores a feature vector $\mathbf{f} \in \mathbb{R}^d$, where $d$ is the channel dimension of the CNN output.
2.  **MLP Decoder:** A standard NeRF MLP with weights $W$. It takes the feature vector $\mathbf{f}$ and the viewing direction $\mathbf{d}$ as input.

**Steps:**
1.  **Feature Volume Generation:** A 3D CNN processes the input data (often derived from multi-view stereo cost volumes or raw images) to produce the 3D feature volume $\mathcal{V}$. The CNN architecture typically involves 3D convolutions and deconvolutions to upsample the volume to the target resolution [S1].
2.  **Ray Sampling:** For a camera ray $\mathbf{r}$, sample $N$ points $\mathbf{x}_i$ along the ray within the bounding volume.
3.  **Feature Trilinear Interpolation:** For each sample point $\mathbf{x}_i$, retrieve the feature vector $\mathbf{f}_i$ from the 3D feature volume $\mathcal{V}$ using trilinear interpolation [G].
4.  **Radiance Field Prediction:**
    *   Compute density $\sigma_i$ and intermediate features using the MLP:
        $$ [\mathbf{h}_i, \sigma_i] = \text{MLP}_{\sigma}(\mathbf{f}_i) $$
    *   Compute color $\mathbf{c}_i$ using the viewing direction $\mathbf{d}$ and intermediate features:
        $$ \mathbf{c}_i = \text{MLP}_{\mathbf{c}}(\mathbf{h}_i, \mathbf{d}) $$
    *   *Note:* The specific MLP architecture follows the standard NeRF formulation where density is predicted from the feature vector and color is predicted from the feature vector concatenated with the view direction [S1].
5.  **Volume Rendering:** Compute the pixel color $\hat{C}(\mathbf{r})$ using the standard differentiable volume rendering equation:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance [S1][S5].
6.  **Optimization:** Minimize the photometric loss between rendered images and ground truth images by backpropagating through the rendering equation and the CNN/MLP parameters [S1].

**Parameters:**
*   **Voxel Resolution:** Not given in the sources.
*   **Feature Dimension ($d$):** Not given in the sources.
*   **MLP Depth/Width:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **High-Frequency Loss:** The single shared MLP structure tends to lose details in high-frequency regions of the image [S1].
*   **Resolution Limits:** Performance is bounded by the resolution of the 3D feature volume; low resolution leads to blurry results [S1].

## Subtopics

## Operating characteristics
*   **Complexity:** Memory usage is dominated by the 3D feature volume, which scales with the cube of the voxel resolution. Computation per ray is dominated by the MLP evaluation and interpolation.
*   **Strengths:** Leverages the inductive bias of CNNs for spatial feature extraction, potentially offering better generalization than pure MLPs for certain structured data [S1].
*   **Limitations:** Tends to lose high-frequency details due to the single shared MLP structure [S1]. Limited by the fixed resolution of the voxel grid, leading to blurriness if resolution is low [S1].
*   **When to use:** When a voxel-based representation is preferred for its efficiency in spatial queries and when the scene benefits from CNN-based feature extraction.
*   **When not to use:** When high-frequency detail preservation is critical, as C-NeRF is noted to lose such details compared to more advanced methods [S1].

## Background
C-NeRF is a follow-up work to Neural Volumes [S1]. It was proposed as a baseline method in comparative studies of volumetric video representation [S1]. The method was re-implemented for comparison in recent research on dynamic MLP maps [S1]. The original code was not released, necessitating re-implementation for evaluation [S1].

## Sources
[S1] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
