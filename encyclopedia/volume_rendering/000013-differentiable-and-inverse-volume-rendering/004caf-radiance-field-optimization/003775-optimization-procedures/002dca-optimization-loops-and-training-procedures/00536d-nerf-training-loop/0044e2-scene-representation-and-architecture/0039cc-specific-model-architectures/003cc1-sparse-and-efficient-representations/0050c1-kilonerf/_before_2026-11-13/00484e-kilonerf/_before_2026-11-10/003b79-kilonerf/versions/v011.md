# KiloNeRF

Also called: KiloNeRF: Speeding up Neural Radiance Fields with Thousands of Tiny MLPs

## Overview
KiloNeRF is a method that decomposes a large, deep Neural Radiance Field (NeRF) into a set of small, shallow NeRFs arranged in a 3D grid to accelerate rendering [S1][S2, unofficial]. It solves the inefficiency of standard NeRFs, which evaluate a single deep network for every sample along a ray, by localizing scene representation to specific spatial regions [S2, unofficial]. Unlike Mega-NeRF, which uses a hierarchical spatial partitioning structure, KiloNeRF employs a uniform 3D grid of lightweight MLPs [S1][S2, unofficial]. It achieves acceleration of up to three orders of magnitude compared to the original NeRF, though it incurs substantial memory overhead due to the deployment of thousands of networks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained standard NeRF (the "teacher") serving as the knowledge source; a 3D grid structure defining the spatial decomposition.
*   **Output:** A set of lightweight MLPs (the "students") arranged in the grid, capable of predicting density $\sigma$ and color $\mathbf{c}$ for points within their assigned grid cell.

**Data Structures:**
*   **3D Grid:** A sparse 3D grid where each active cell contains a reference to a tiny MLP [S1][S3, unofficial].
*   **Tiny MLP:** A shallow neural network with significantly fewer layers and parameters than the original NeRF [S2, unofficial].

**Steps:**
1.  **Teacher Training:** Train a standard, deep NeRF on the scene data to convergence [S1].
2.  **Spatial Decomposition:** Divide the 3D space into a grid of cells [S2, unofficial].
3.  **Student Initialization:** Initialize a tiny MLP for each grid cell [S1].
4.  **Distillation:** Distill the knowledge from the teacher NeRF into the student MLPs. This involves training the tiny MLPs to approximate the teacher's output for points falling within their respective grid cells [S1].
5.  **Surface Localization (Optional/Enhanced):** Use a cached sparse 3D grid to quickly locate surface geometry, allowing rays to skip empty space and only query relevant MLPs [S3, unofficial].
6.  **Rendering:** For a given ray, determine which grid cells it intersects. Query the corresponding tiny MLPs for density and color at sample points within those cells. Compute the final pixel color using standard volumetric rendering [S1][S2, unofficial].

**Formulas:**
The rendering equation remains the standard volume rendering integral, but the function $F_\Theta$ is replaced by a set of functions $\{F_{\theta_i}\}$, where $\theta_i$ are the parameters of the tiny MLP in grid cell $i$:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\sigma_i = F_{\theta_i}(\mathbf{x}_i, \mathbf{d}_i).\sigma$ and $\mathbf{c}_i = F_{\theta_i}(\mathbf{x}_i, \mathbf{d}_i).\mathbf{c}$ [S1][S2, unofficial].

**Parameters:**
*   **Grid Resolution:** Not given in the sources.
*   **Tiny MLP Depth/Width:** Significantly smaller than the original NeRF, but exact dimensions are not given in the sources [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** Deploying a large number of MLPs incurs substantial memory overhead [S1].
*   **Dynamic Parameter Loading:** During rendering, a ray intersects different sets of NeRFs, each requiring a different number of samples. This dynamic parameter loading requires special CUDA kernels for proper speedup, as static loading is inefficient [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Acceleration of up to three orders of magnitude compared to the original NeRF [S1].
*   **Memory Complexity:** Substantial memory overhead due to the deployment of thousands of MLPs [S1].
*   **Strengths:** Significant rendering speedup; efficient representation of local scene details [S1][S2, unofficial].
*   **Limitations:** High memory usage; requires specialized rendering kernels to handle dynamic parameter loading efficiently [S1][S2, unofficial].
*   **When to use:** When real-time or near-real-time rendering of large scenes is required and memory is not the primary constraint [S1].
*   **When not to use:** When memory is severely constrained or when the scene does not benefit from spatial decomposition (e.g., very small scenes) [S1][S2, unofficial].

## Background
KiloNeRF was introduced by Reiser et al. in 2021 [S1]. It was developed to address the computational inefficiency of standard NeRFs, which sample the entire ray even though only a small fraction of samples contribute to the final image [S1]. The method builds on the idea of spatial decomposition, similar to Neural Sparse Voxel Fields (NSVF), but uses tiny MLPs instead of feature vectors [S2, unofficial].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
