# Spatially Encoded Radiance Fields

Also called: Sparse Radiance Fields, Voxel-based NeRF

## Overview
Spatially Encoded Radiance Fields (SERF) simplify the implicit radiance function by spatially decomposing it into smaller functions to reduce the computational complexity load on the Multi-Layer Perceptron (MLP) [S1]. By storing feature vectors in a 3D voxel grid, the method divides volumetric model information into local batches, allowing equal or higher rendering quality than coordinate-based NeRF with a smaller or faster-decoding MLP [S1]. It sets itself apart from standard NeRF by replacing the heavy coordinate-to-color mapping with a lookup-and-decode pipeline, and from sibling sparse methods by explicitly defining the grid-based feature storage mechanism that enables this acceleration [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of calibrated images with known camera poses; optionally, time $t$ for dynamic scenes.
*   **Output:** A rendered image (color $\mathbf{c}$) and depth, derived from the radiance field parameters.

**Data Structures:**
1.  **Density Volume Grid ($\mathbf{V}_\sigma$):** A 3D tensor $\mathbb{R}^{H \times W \times D}$ storing volume density $\sigma$ [S2]. In sparse implementations, this may be stored in a hash table or octree structure to save memory [S1].
2.  **Feature Volume Grid ($\mathbf{V}_a$):** A 3D tensor $\mathbb{R}^{H \times W \times D \times F}$ storing latent feature vectors $f$ of dimension $F$ [S2]. These features encode appearance and geometry information [S1].
3.  **Decoder Network ($S_\Theta$):** A small MLP that takes the interpolated feature vector as input and outputs the final color $\mathbf{c}$ [S2].

**Steps:**
1.  **Grid Initialization:** Initialize $\mathbf{V}_\sigma$ and $\mathbf{V}_a$ with learnable parameters. For sparse variants, initialize the hash map or octree structure [S1].
2.  **Query Point Sampling:** For a ray passing through the scene, sample points $\mathbf{x}$ along the ray [S1].
3.  **Feature Interpolation:** Retrieve the feature vector $f$ and density $\sigma$ at $\mathbf{x}$ from the grids using trilinear interpolation [S2]:
    $$ f(\mathbf{x}) = \text{interp}(\mathbf{x}, \mathbf{V}_a) $$
    $$ \sigma(\mathbf{x}) = \text{interp}(\mathbf{x}, \mathbf{V}_\sigma) $$
    [S2].
4.  **Color Decoding:** Pass the interpolated feature vector $f(\mathbf{x})$ through the decoder MLP $S_\Theta$ to predict the view-dependent color $\mathbf{c}$ [S2]:
    $$ \mathbf{c}(\mathbf{x}) = S_\Theta(f(\mathbf{x})) $$
    [S2].
5.  **Volume Rendering:** Compute the final pixel color using standard volume rendering integration over the sampled points, combining $\sigma(\mathbf{x})$ and $\mathbf{c}(\mathbf{x})$ [S1].
6.  **Optimization:** Update $\mathbf{V}_\sigma$, $\mathbf{V}_a$, and $S_\Theta$ parameters via gradient descent to minimize the difference between rendered and ground-truth images [S1].

**Parameters:**
*   Grid Resolution ($H, W, D$): Not given in the sources.
*   Feature Dimension ($F$): Not given in the sources.
*   Decoder Size: Typically smaller than standard NeRF MLPs due to the pre-computed features [S1].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Dense grids scale cubically with resolution. Sparse representations (e.g., hash grids) are required for high resolutions to avoid out-of-memory errors [S1].
*   **Interpolation Artifacts:** Trilinear interpolation may introduce blurring compared to nearest-neighbor or higher-order methods, potentially affecting high-frequency detail [S1].
*   **Boundary Conditions:** Points outside the grid bounds must be handled (e.g., clamping or treating as transparent) [G].

## Subtopics

### Efficient EndoNeRF scene representation
A voxel-grids-based scene representation that replaces coordinate-based MLPs with explicit density and feature volume grids to accelerate optimization, combined with a canonical field and time-dependent deformation field for dynamic surgical scenes [S2]. It differs from static SERF by modeling dynamic scenes via warping a canonical radiance field with a time-dependent deformation field [S2]. Choose this for dynamic endoscopic/surgical video reconstruction.

## Operating characteristics
*   **Time Complexity:** Faster inference and optimization than coordinate-based NeRF because the MLP is smaller and operates on lower-dimensional feature vectors rather than raw coordinates [S1].
*   **Memory Complexity:** Scales with the number of voxels. Dense grids are $O(H \times W \times D)$, while sparse grids reduce this significantly [S1].
*   **Strengths:** Faster optimization convergence; reduced MLP complexity; suitable for real-time or near-real-time rendering with sufficient grid resolution [S1].
*   **Limitations:** Memory intensive for high-resolution dense grids; interpolation may smooth out fine details compared to pure MLP representations [S1].
*   **When to use:** When training speed and inference latency are critical, and the scene can be adequately represented by a voxel grid [S1].
*   **When not to use:** For scenes with extremely high-frequency details that cannot be captured by the grid resolution, or when memory is severely constrained and sparse structures are not feasible [S1].

## Background
The concept of spatially decomposing the radiance function to reduce MLP complexity was demonstrated by Yu et al. [S1]. Müller et al. developed Instant-NeRF, a multi-resolution feature voxel grid implementation [S1]. The approach addresses the slow inference speed of standard NeRF MLPs [S1]. Efficient EndoNeRF extends this to dynamic surgical scenes using canonical fields and deformation fields [S2].

## Sources
[S1] ADVANCEMENTS IN RADIANCE FIELD TECHNIQUES FOR VOLUMETRIC VIDEO GENERATION: A TECHNICAL OVERVIEW, https://hal.science/hal-05049100v1, fetched 2026-10-26
[S2] Efficient EndoNeRF reconstruction and its application for data-driven surgical simulation., https://doi.org/10.1007/s11548-024-03114-1, fetched 2026-10-26
