# KiloNeRF

Also called: KiloNeRF: Speeding up Neural Radiance Fields with Thousands of Tiny MLPs

## Overview
KiloNeRF is a spatial decomposition method that accelerates neural radiance field rendering by replacing a single large deep MLP with thousands of small, shallow MLPs arranged in a 3D grid [S1][S2, unofficial]. It solves the high per-point computation cost of standard NeRF by distilling a teacher network's knowledge into these lightweight student networks, achieving acceleration of up to three orders of magnitude [S1]. Unlike sibling methods that may use caching or octrees, KiloNeRF relies on explicit spatial partitioning into tiny MLPs, requiring dynamic parameter loading during rendering [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A pre-trained standard NeRF (teacher) defined by parameters $\theta_{teacher}$, and a 3D grid structure defining the spatial partition.
*   Output: A set of lightweight MLPs (students) with parameters $\{\theta_i\}$, one for each active grid cell, capable of predicting density $\sigma$ and color $\mathbf{c}$ for queries within their respective cells.

**Data Structures:**
*   **3D Grid:** A sparse 3D grid where each cell corresponds to a region of the scene. The grid stores indices or pointers to the specific tiny MLP assigned to that cell.
*   **Tiny MLPs:** Each student network is a shallow Multi-Layer Perceptron. The exact width and depth are not specified in the sources, but they are significantly smaller than the original NeRF MLP [S1][S2, unofficial].
*   **Cached Sparse 3D Grid:** Used to quickly locate the surface and determine which tiny MLPs are relevant for a given ray [S3, unofficial].

**Training Procedure (Teacher-Student Distillation):**
1.  **Teacher Training:** Train a standard, deep NeRF on the scene data to convergence. Let this network be $f_{\theta_{teacher}}(\mathbf{x}, \mathbf{d})$.
2.  **Grid Initialization:** Define a 3D grid partitioning the scene space. Assign a tiny MLP $f_{\theta_i}$ to each grid cell $i$.
3.  **Distillation:** Optimize the parameters $\theta_i$ of each tiny MLP to mimic the output of the teacher network for points falling within its corresponding grid cell. The loss function minimizes the difference between the student's output and the teacher's output for sampled points.
    *   For a query point $\mathbf{x}$ in cell $i$ and direction $\mathbf{d}$:
        $$ \mathcal{L}_i = \| f_{\theta_i}(\mathbf{x}, \mathbf{d}) - f_{\theta_{teacher}}(\mathbf{x}, \mathbf{d}) \|^2 $$
    *   The total loss is the sum over all active cells.
4.  **Final Model:** The final radiance field is represented by the collection of these tiny MLPs.

**Rendering Procedure:**
1.  **Ray Casting:** For each pixel, cast a ray $\mathbf{r}$ through the scene.
2.  **Spatial Lookup:** Use the cached sparse 3D grid to identify which grid cells the ray intersects.
3.  **Dynamic Parameter Loading:** As the ray traverses different cells, load the parameters of the corresponding tiny MLP into the GPU memory/cache. This is a dynamic process because different rays traverse different sets of MLPs [S2, unofficial].
4.  **Evaluation:** For each sample point along the ray, query the currently active tiny MLP to get $\sigma$ and $\mathbf{c}$.
5.  **Volume Rendering:** Compute the pixel color using the standard volume rendering integral:
    $$ \mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$.

**Parameters:**
*   Grid resolution: Not given in the sources.
*   Tiny MLP size: Not given in the sources, but described as "small" and "shallow" [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** Deploying a large number of MLPs incurs substantial memory overhead [S1].
*   **Dynamic Loading:** During rendering, a ray goes through different sets of NeRFs taking different numbers of samples, requiring dynamic parameter loading. This necessitates special CUDA kernels for proper speedup [S2, unofficial].
*   **Boundary Artifacts:** Points near grid cell boundaries may experience discontinuities if the tiny MLPs are not trained to be consistent across boundaries. This is not explicitly addressed in the provided notes.

## Operating characteristics
*   **Time Complexity:** Acceleration of up to three orders of magnitude compared to the original NeRF [S1].
*   **Memory Complexity:** Substantial memory overhead due to the deployment of thousands of MLPs [S1].
*   **Strengths:** Significant speedup in rendering time; allows for real-time or near-real-time view synthesis.
*   **Limitations:** High memory usage; requires complex dynamic parameter loading logic for optimal performance [S2, unofficial].
*   **When to use:** When rendering speed is critical and memory resources are sufficient to store thousands of small networks.
*   **When not to use:** When memory is constrained or when the scene geometry is too complex for a simple grid partition to capture efficiently without excessive cell count.

## Background
KiloNeRF was introduced by Reiser et al. in 2021 [S1]. It is part of the broader effort to accelerate NeRF rendering by reducing the computational cost per sample point, alongside methods like SNeRG, PlenOctree, and MobileNeRF [S3, unofficial]. The method builds on the observation that surfaces occupy only a small fraction of the 3D space, and thus, a large NeRF is inefficient for representing small portions of space [S2, unofficial].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
