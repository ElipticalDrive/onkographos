# C-NeRF
Also called: Volume Basis
## Overview
C-NeRF is a voxel-based radiance field optimization method that adopts a 3D convolutional neural network to produce feature volumes, which are then decoded by a NeRF-like MLP to predict radiance fields [S2]. It improves upon its predecessor Neural Volumes by leveraging 3D convolutions for volumetric data, though it still tends to lose high-frequency details due to its single shared MLP structure [S2].
## How it works
**Inputs and Outputs:**
- Input: Multi-view images and camera poses [G].
- Output: A differentiable radiance field representation consisting of a 3D feature volume and a decoding MLP, allowing for novel view synthesis and volume rendering [S2].

**Data Structures:**
1. **3D Feature Volume:** A 3D grid of tensors produced by a 3D CNN [S2].
2. **Decoding MLP:** A Multi-Layer Perceptron that takes spatial coordinates and the sampled feature vector from the 3D volume as input, and outputs density ($\sigma$) and color ($\mathbf{c}$) [S2].

**Method Steps:**
1. **3D CNN Encoding:** A 3D Convolutional Neural Network processes the input data to produce a 3D feature volume [S2]. This step replaces the implicit coordinate mapping of standard NeRF with an explicit volumetric representation [G].
2. **Ray Sampling:** For each ray $\mathbf{r}$, sample $N$ points along the ray within the scene bounding box [G].
3. **Feature Interpolation:** For each sample point, trilinearly interpolate the feature vector from the 3D feature volume based on the point's spatial coordinates [G].
4. **Radiance Field Prediction:** Pass the spatial coordinates and the interpolated feature vector into the NeRF-like MLP [S2]. The MLP outputs the density $\sigma_i$ and color $\mathbf{c}_i$ for each sample point $i$ [S2].
5. **Volume Rendering:** Compute the final pixel color $\hat{C}(\mathbf{r})$ using the standard differentiable volume rendering equation [G]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance, and $\delta_i$ is the distance between sample points [G].
6. **Optimization:** Minimize the photometric loss between rendered images and ground truth images using gradient descent [G]. Gradients are backpropagated through the rendering equation and the MLP [S2]. The 3D CNN parameters are updated if the network is trained end-to-end from raw inputs [S2].

**Parameters:**
- **Voxel Resolution:** Not given in the sources.
- **CNN Architecture:** Not given in the sources.
- **MLP Architecture:** Not given in the sources.

**Edge Cases and Pitfalls:**
- **High-Frequency Detail Loss:** C-NeRF tends to lose details in high-frequency regions of the image compared to more advanced methods, attributed to the limitations of its single shared MLP model structure [S2].
- **Memory Usage:** Storing a 3D feature volume requires significant memory, scaling with the cube of the resolution [G].
## Subtopics

### [MVSNeRF](11941-mvsnerf.md)
Fast, generalizable radiance field reconstruction from multi-view stereo that learns a data prior to bypass per-scene optimization. Uses a pretrained CNN to extract 2D features, maps them to a 3D voxelized cost volume via plane sweeping, and uses a pretrained 3D CNN for a 3D neural encoding volume. Choose when per-scene optimization is too slow and multi-view inputs are available.

## Operating characteristics
**Time and Memory Complexity:**
- **Memory:** Dominated by the storage of the 3D feature volume and the CNN/MLP parameters [S2].
- **Time:** Training time is significantly faster than coordinate-based NeRF due to the efficiency of 3D convolutions and parallelizable voxel access [S2]. However, specific complexity formulas are not provided [S2].

**Measured Performance:**
- Qualitative results show it produces sharper images than Neural Volumes but still suffers from blurring in high-frequency regions compared to methods using dynamic representations [S2].

**Strengths and Limitations:**
- **Strengths:** Faster training than MLP-only NeRF; better detail preservation than Neural Volumes; explicit volumetric representation allows for efficient 3D operations [S2].
- **Limitations:** Loses high-frequency details due to its single shared MLP model structure [S2].

**When to Use:**
- When faster training than standard NeRF is required and the scene has moderate complexity [S2].
- When an explicit volumetric representation is preferred for downstream 3D processing [S2].

**When Not to Use:**
- When high-frequency detail preservation is critical [S2].
- When memory is constrained and high resolution is needed [G].
## Background
C-NeRF was introduced as a follow-up work to Neural Volumes [S2]. It is part of the evolution of voxel-based radiance field representations, aiming to balance the efficiency of voxel grids with the expressive power of neural networks for volumetric video representation and novel view synthesis [S2].
## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
[S2] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
