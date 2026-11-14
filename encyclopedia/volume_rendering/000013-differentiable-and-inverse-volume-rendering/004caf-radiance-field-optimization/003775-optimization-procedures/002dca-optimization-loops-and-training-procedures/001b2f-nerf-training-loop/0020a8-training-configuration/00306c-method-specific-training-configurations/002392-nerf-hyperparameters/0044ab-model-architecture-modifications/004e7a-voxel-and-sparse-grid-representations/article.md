# Voxel and Sparse Grid Representations

Also called: Grid-based Radiance Fields, Hash Encoding, Tri-Plane Representations

## Overview
Voxel and Sparse Grid Representations replace the continuous Multi-Layer Perceptron (MLP) function in standard Neural Radiance Fields with discrete data structures (dense grids, sparse grids, or hash tables) to store volume density and view-dependent color features [S1]. This substitution eliminates the computational bottleneck of evaluating fully-connected layers for every sample point, enabling real-time rendering by directly interpolating grid values [S1]. Unlike sibling methods that modify input conditioning or output heads, this family fundamentally changes the spatial representation from implicit neural functions to explicit, queryable discrete structures.

## Shared principles
The core mechanism involves defining a spatial data structure that maps 3D coordinates $\mathbf{x}$ to feature vectors. For any query point $\mathbf{x}_k$ along a ray, the method retrieves features via interpolation (e.g., trilinear) or hashing. The retrieved features are typically decomposed into:
1.  **Density ($\sigma$):** A scalar value used to compute opacity $\alpha_i = 1 - e^{-\sigma_i \delta_i}$ and transmittance weights $w_i$ for volume rendering [S1].
2.  **Appearance Features:** Vectors representing color, often encoded as Spherical Harmonic (SH) coefficients or separate diffuse/specular components, which are projected onto the viewing direction $\mathbf{d}$ to produce color $\mathbf{c}_k$ [S1].

The rendering integral remains the standard differentiable volume rendering equation:
$$
\hat{C}(r) = \sum_{i=1}^{N} w_i \mathbf{c}_i, \quad w_i = T_i \alpha_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\mathbf{c}_i$ is computed from the grid features at $\mathbf{x}_i$ and $\mathbf{d}$ [S1].

## Subtopics

### [Voxel-based Radiance Fields](11470-voxel-based-radiance-fields.md)
Replaces the MLP with two dense voxel grids: a density grid $G_\sigma$ and an SH coefficient grid $G_{sh}$. It is chosen for its simplicity and memory efficiency compared to large MLPs, though it scales cubically with resolution.

### [Neural Multi-View Image Reconstruction with Hash Encoding](13013-neural-multi-view-image-reconstruction-with-hash-encoding.md)
Uses a hash table for spatial feature storage combined with an SDF for geometry. It is distinguished by explicitly optimizing SDF for geometry, avoiding noise in unconstrained density fields, and using hash encoding for high-frequency detail storage.

### [G-NeLF](13480-g-nelf.md)
A grid-based NeLF that predicts ray color directly without point-wise density modeling, using a hash-based multi-resolution tri-plane and an LSTM decoder. It is chosen for its extremely low parameter count (~10% of comparable grid-based NeRFs) and suitability for training on few views without pseudo-view synthesis.

### SNeRG Rendering Pipeline
A real-time rendering pipeline that stores a NeRF model in a sparse 3D voxel grid, using an indirection grid to skip empty regions and a deferred MLP to compute view-dependent color from accumulated diffuse and specular features [S1].
- **Step:** Raymarch along $r(t) = o + td$. For occupied voxels, set step size $\delta_i$ equal to the voxel width [S1].
- **Formula:** $w_i = \prod_{j=1}^{i-1} (1 - \alpha_j) \alpha_i$, where $w_i$ is the accumulated weight for sample $i$ [S1].
- **Mechanism:** Concatenate accumulated diffuse color $C_d$, accumulated specular feature $F_s$, and positional encoding $PE(\mathbf{d})$ of the ray's view direction $\mathbf{d}$. Pass through a lightweight deferred MLP $\Phi$ to produce a view-dependent residual color [S1]:
  $$
  \boldsymbol{C}(\boldsymbol{r}) = \boldsymbol{C}_d + \Phi (\boldsymbol{C}_d, \boldsymbol{F}_s, PE(\boldsymbol{d}))
  $$
  [S1]

## Background
The shift from MLPs to grid-based representations was driven by the need for real-time rendering on web platforms and consumer hardware. SNeRG [13] and MERF [30] set benchmarks for real-time rendering by precomputing and storing radiance fields in sparse 3D voxel grids, using indirection grids to enhance rendering by skipping empty regions [S1]. City-on-Web extended these concepts to large-scale scenes, leveraging hybrid low-resolution sparse voxels and high-resolution triplanes to optimize the balance between performance and memory efficiency [S1].

## Sources
[S1] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
