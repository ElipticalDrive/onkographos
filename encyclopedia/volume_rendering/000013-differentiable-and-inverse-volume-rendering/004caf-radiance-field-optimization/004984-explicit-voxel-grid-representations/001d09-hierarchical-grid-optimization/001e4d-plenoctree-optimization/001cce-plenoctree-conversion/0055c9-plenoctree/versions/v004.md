# PlenOctree


## Overview
PlenOctree is a sparse octree-based 3D representation that enables real-time rendering of Neural Radiance Fields (NeRFs) by pre-tabulating view-dependent radiance into a voxel structure [S2, unofficial]. Unlike naive NeRF rendering which requires dense sampling and neural network inference for every sample dependent on viewing direction, PlenOctrees remove the need for neural network queries at test time by storing precomputed spherical harmonic coefficients [S2, unofficial]. It achieves inference speeds approximately 3000 times faster than the original NeRF implementation [S1].

## How it works
**Inputs:** A trained NeRF-SH model (MLP predicting density $\sigma$ and spherical harmonic coefficients for color) or a standard NeRF model; training images.
**Outputs:** A sparse octree data structure storing density and SH coefficients at leaf nodes.

**Data Structures:**
The PlenOctree is a sparse voxel-based octree [S2, unofficial].
- **Node:** Internal nodes contain pointers to children and metadata for culling (e.g., bounding box, transmissivity).
- **Leaf:** Stores the radiance properties for the voxel.
  - **Density ($\sigma$):** A scalar value representing volume density.
  - **Spherical Harmonic Coefficients ($c_l^m$):** A set of coefficients representing the view-dependent color function. The number of bands $L$ determines the resolution of view-dependent effects.
  - **Layout:** Not given in the sources.

**Steps:**

1.  **Train NeRF-SH:** Train a NeRF model to predict spherical harmonic coefficients of the color function instead of directly predicting color values [S1]. The color $c(\mathbf{r})$ for a ray direction $\mathbf{r} = (\theta, \phi)$ is calculated by summing the weighted spherical harmonic bases evaluated at the corresponding ray direction [S2, unofficial].
2.  **Voxelization and Sampling:** Densely sample the NeRF-SH model in the volume around the target object [S2, unofficial]. Voxelize the scene during octree construction [S1].
3.  **Tabulation:** For each voxel, tabulate the density $\sigma$ and the spherical harmonic coefficients derived from the MLP’s colors [S1].
    - If starting from a standard NeRF (not NeRF-SH), perform Monte Carlo estimations of the spherical harmonic components to derive the coefficients [S1].
4.  **Pruning:** Eliminate voxels with low transmissivity to create a sparse structure [S1]. This allows volume rendering to skip empty space [S3, unofficial].
5.  **Optimization (Fine-tuning):** Optimize the values stored in the PlenOctree directly with the training images to improve quality, as the rendering procedure remains differentiable [S2, unofficial]. This is a fast fine-tuning procedure relative to NeRF training [S1].

**Formulas:**
The color $c(\mathbf{r})$ is modeled using spherical harmonics:
$$ c(\mathbf{r}) = \sum_{l=0}^{L} \sum_{m=-l}^{l} c_l^m Y_l^m(\theta, \phi) $$
where $Y_l^m$ are the spherical harmonic basis functions, $\theta, \phi$ are the spherical coordinates of the ray direction, and $c_l^m$ are the coefficients stored in the PlenOctree leaf [S2, unofficial].

**Parameters:**
- **Spherical Harmonic Bands ($L$):** Not given in the sources.
- **Octree Depth:** Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Not given in the sources.
- **Boundaries:** Not given in the sources.
- **Degenerate Input:** Not given in the sources.

## Subtopics

## Operating characteristics
- **Time Complexity:** Rendering is significantly faster than standard NeRF. Inference times are approximately 3000 times faster than the original NeRF implementation [S1].
- **Memory Complexity:** Uses a sparse octree to reduce memory requirements compared to dense grids [S2, unofficial].
- **Measured Performance:** Renders 800x800 images at 167.68 FPS on an NVIDIA V100 GPU [S2, unofficial]. This is over 3000 times faster than conventional NeRFs (approx. 30 seconds per image) [S2, unofficial].
- **Scaling:** Not given in the sources.
- **Strengths:** Real-time performance without sacrificing quality; preserves ability to perform free-viewpoint rendering of scenes with arbitrary geometry and view-dependent effects; amenable to modern web technologies [S2, unofficial].
- **Limitations:** Requires an initial training phase (NeRF-SH or Monte Carlo estimation) and fine-tuning; quality depends on the number of SH bands used [S2, unofficial].
- **When to use:** When real-time rendering of view-dependent NeRFs is required [S2, unofficial].
- **When not:** When interactive training is preferred over pre-tabulated rendering; when extreme view-dependent effects require very high SH bands (memory intensive) [S2, unofficial].

## Background
PlenOctrees were introduced by Alex Yu et al. in March 2021 [S1]. The method is named after plenoptic functions [S2, unofficial]. It was developed to overcome the slowness of naive NeRF rendering, which requires dense sampling and neural network inference for every sample dependent on viewing direction [S2, unofficial]. The approach was concurrent with other acceleration methods like FastNeRF and KiloNeRF [S1][S3, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
