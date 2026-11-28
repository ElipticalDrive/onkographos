# C-NeRF
Also called: Neural Volumes follow-up
## Overview
C-NeRF is a voxel-based radiance field optimization method that replaces the coordinate-based MLP of standard NeRF with a 3D convolutional neural network to produce feature volumes, which are then decoded by a NeRF-like MLP to predict radiance fields [S2]. It is designed to improve upon the blurry results of its predecessor, Neural Volumes, by leveraging the inductive bias of 3D convolutions for volumetric data [S2]. Unlike coordinate-based methods that suffer from high-frequency detail loss due to single shared MLP structures, C-NeRF operates on a discrete voxel grid, enabling faster training and inference for volumetric video representation [S2].

## How it works
**Inputs and Outputs:**
- Input: A set of multi-view images (or a single view for reconstruction) and camera poses.
- Output: A differentiable radiance field representation consisting of a 3D feature volume and a decoding MLP, allowing for novel view synthesis and volume rendering.

**Data Structures:**
1.  **3D Feature Volume:** A 3D grid of tensors. Each voxel contains a feature vector (dimension not explicitly specified in sources, but implied to be fixed per voxel). The resolution is determined by the scene bounding box and chosen grid size.
2.  **Decoding MLP:** A Multi-Layer Perceptron that takes spatial coordinates and the sampled feature vector from the 3D volume as input, and outputs density ($\sigma$) and color ($\mathbf{c}$).

**Method Steps:**
1.  **3D CNN Encoding:** A 3D Convolutional Neural Network processes the input data (or a cost volume derived from multi-view stereo) to produce a 3D feature volume [S2]. This step replaces the implicit coordinate mapping of standard NeRF with an explicit volumetric representation.
2.  **Ray Sampling:** For each ray $\mathbf{r}$, sample $N$ points along the ray within the scene bounding box.
3.  **Feature Interpolation:** For each sample point, trilinearly interpolate the feature vector from the 3D feature volume based on the point's spatial coordinates.
4.  **Radiance Field Prediction:** Pass the spatial coordinates and the interpolated feature vector into the NeRF-like MLP. The MLP outputs the density $\sigma_i$ and color $\mathbf{c}_i$ for each sample point $i$.
5.  **Volume Rendering:** Compute the final pixel color $\hat{C}(\mathbf{r})$ using the standard differentiable volume rendering equation [44-radiance-field-optimization.md]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance, and $\delta_i$ is the distance between sample points.
6.  **Optimization:** Minimize the photometric loss between rendered images and ground truth images using gradient descent. Gradients are backpropagated through the rendering equation and the MLP. The 3D CNN parameters are also updated if the network is trained end-to-end from raw inputs [S2].

**Parameters:**
-   **Voxel Resolution:** Not given in the sources.
-   **CNN Architecture:** Not given in the sources.
-   **MLP Architecture:** Not given in the sources.

**Edge Cases and Pitfalls:**
-   **High-Frequency Detail Loss:** C-NeRF tends to lose details in high-frequency regions of the image compared to more advanced methods, attributed to the limitations of its single shared MLP model structure combined with the voxel resolution [S2].
-   **Memory Usage:** Storing a 3D feature volume requires significant memory, scaling with the cube of the resolution.

## Subtopics
## Operating characteristics
**Time and Memory Complexity:**
-   **Memory:** Dominated by the storage of the 3D feature volume and the CNN/MLP parameters. The volume size scales with $R^3$ where $R$ is the resolution.
-   **Time:** Training time is significantly faster than coordinate-based NeRF due to the efficiency of 3D convolutions and parallelizable voxel access. However, specific complexity formulas are not provided.

**Measured Performance:**
-   C-NeRF is evaluated on the D-NeRF dataset [S1].
-   It achieves higher PSNR and SSIM than Neural Volumes but lower than some newer factorization methods like HexPlane [S1].
-   Qualitative results show it produces sharper images than Neural Volumes but still suffers from blurring in high-frequency regions compared to methods using dynamic representations [S2].

**Strengths and Limitations:**
-   **Strengths:** Faster training than MLP-only NeRF; better detail preservation than Neural Volumes; explicit volumetric representation allows for efficient 3D operations.
-   **Limitations:** Loses high-frequency details; memory intensive for high resolutions; less flexible than implicit representations for complex geometries [S2].

**When to Use:**
-   When faster training than standard NeRF is required and the scene has moderate complexity.
-   When an explicit volumetric representation is preferred for downstream 3D processing.

**When Not to Use:**
-   When high-frequency detail preservation is critical.
-   When memory is constrained and high resolution is needed.

## Background
C-NeRF was introduced as a follow-up work to Neural Volumes [S2]. It was proposed to address the blurriness and lack of detail in Neural Volumes by adopting a 3D CNN to produce feature volumes, which are then decoded by a NeRF-like MLP [S2]. The method is part of the evolution of voxel-based radiance field representations, aiming to balance the efficiency of voxel grids with the expressive power of neural networks. It is used in the context of volumetric video representation and novel view synthesis [S2].

## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
[S2] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
