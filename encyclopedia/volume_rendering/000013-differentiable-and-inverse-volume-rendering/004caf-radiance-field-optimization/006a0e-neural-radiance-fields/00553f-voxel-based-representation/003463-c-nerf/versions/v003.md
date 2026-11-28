# C-NeRF
Also called: Volume Basis
## Overview
C-NeRF is a voxel-based radiance field representation that uses a 3D convolutional neural network (CNN) to generate feature volumes, which are then decoded by a NeRF-like multi-layer perceptron (MLP) to predict density and color [S2]. It is distinguished from its parent Voxel-based Representation by explicitly learning spatial features via convolutions rather than relying solely on interpolation of static grid values [S2]. It serves as a baseline for dynamic scene representation, often referred to as "Volume Basis" when decomposing 4D volumes into shared 3D components [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ (for color).
*   **Output:** Density $\sigma$ and view-dependent color $\mathbf{c}$.

**Data Structures:**
1.  **Feature Volume:** A 3D tensor $\mathcal{V} \in \mathbb{R}^{H \times W \times D \times C}$, where $H, W, D$ are the grid dimensions and $C$ is the feature channel dimension. This volume is the output of a 3D CNN encoder.
2.  **Decoder MLP:** A small multi-layer perceptron that takes the sampled feature vector and positional encoding as input.

**Steps:**
1.  **Feature Volume Generation:** A 3D CNN encoder processes input data (e.g., multi-view images or canonical space representations) to produce the feature volume $\mathcal{V}$ [S2].
2.  **Ray Sampling:** For a camera ray $\mathbf{r}$, sample $N$ points $\mathbf{x}_i$ along the ray [Parent].
3.  **Feature Sampling:** For each point $\mathbf{x}_i$, retrieve the feature vector $\mathbf{f}_i$ from $\mathcal{V}$ using trilinear interpolation [Parent].
4.  **Positional Encoding:** Apply positional encoding to $\mathbf{x}_i$ and $\mathbf{d}$ (if view-dependent). Let $\gamma(\mathbf{x}_i)$ be the encoded spatial position.
5.  **Decoding:** Concatenate $\mathbf{f}_i$ with $\gamma(\mathbf{x}_i)$ and optionally $\mathbf{d}$. Pass through the MLP:
    $$
    [\sigma_i, \mathbf{c}_i] = \text{MLP}([\mathbf{f}_i, \gamma(\mathbf{x}_i), \mathbf{d}])
    $$
    where $\sigma_i$ is the volume density and $\mathbf{c}_i$ is the color [S2].
6.  **Volume Rendering:** Integrate $\sigma_i$ and $\mathbf{c}_i$ using the standard volume rendering equation [Parent]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$

**Parameters:**
*   **Grid Resolution:** Not specified in sources for the base C-NeRF implementation, but typically defined by the CNN output size [S2].
*   **Feature Channels ($C$):** Not specified in sources [S2].

**Edge Cases and Pitfalls:**
*   **High-Frequency Loss:** The single shared MLP structure tends to lose details in high-frequency regions of the image [S2].
*   **Deformation Difficulty:** When applied to dynamic scenes without explicit deformation fields (as in D-NeRF), complex scene motions are difficult to learn [S2].

## Subtopics

### [MVSNeRF](11941-mvsnerf.md)
MVSNeRF is a method for fast, generalizable radiance field reconstruction from multi-view stereo that bypasses per-scene optimization by learning a data prior. It extracts 2D image features, maps them to a 3D voxelized cost volume via plane sweeping, and uses a pretrained 3D CNN to generate a 3D neural encoding volume. Unlike its parent C-NeRF, which uses a 3D CNN to produce feature volumes and appends a NeRF-like MLP, MVSNeRF constructs perspective frustum volumes for small-baseline rendering and jointly optimizes the 3D feature volume and the NeRF MLP during inference.

## Operating characteristics
*   **Performance:** On the D-NeRF dataset with basis number $R=24$, the "Volume Basis" variant (equivalent to C-NeRF's factorization) achieves PSNR 30.329, SSIM 0.962, LPIPS 0.051, and Training Time 14m 36s [S1].
*   **Strengths:** Efficient compared to pure MLP-based methods like DyNeRF; avoids the blurry results of low-resolution Neural Volumes [S2].
*   **Limitations:** Tends to lose details in high-frequency regions due to the shared MLP structure [S2]. Struggles with complex scene motions if deformation is not explicitly modeled [S2].
*   **Scaling:** Performance scales with the basis number $R$ (or grid resolution), with higher $R$ generally improving quality but increasing training time [S1].

## Background
C-NeRF is a follow-up work to Neural Volumes [35], adopting a 3D CNN to produce feature volumes and appending a NeRF-like MLP to predict radiance fields [S2]. It is used as a baseline method in comparisons for dynamic volumetric video representation [S2]. The term "Volume Basis" is used in HexPlane to describe this specific factorization of 4D volumes as a weighted summation of shared 3D volumes [S1].

## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
[S2] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
