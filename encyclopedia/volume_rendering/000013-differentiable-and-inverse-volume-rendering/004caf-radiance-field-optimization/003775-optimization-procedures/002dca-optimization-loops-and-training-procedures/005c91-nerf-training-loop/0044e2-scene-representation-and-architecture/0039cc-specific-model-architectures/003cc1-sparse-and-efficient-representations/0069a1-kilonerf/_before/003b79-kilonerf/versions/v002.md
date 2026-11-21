# KiloNeRF

Also called: KiloNeRF: Speeding up Neural Radiance Fields with Thousands of Tiny MLPs

## Overview
KiloNeRF is a method that decomposes a large, deep Neural Radiance Field (NeRF) into a set of small, shallow NeRFs arranged in a 3D grid to reduce per-point computational cost [S1][S2, S3, unofficial]. It solves the inefficiency of standard volumetric rendering, where a single monolithic network processes every sample along a ray regardless of local scene complexity [S2, unofficial]. Unlike siblings that use discrete voxel features (e.g., SRF) or hierarchical trees (e.g., PlenOctree), KiloNeRF retains the continuous MLP representation but distributes it spatially, achieving acceleration of up to three orders of magnitude compared to the original NeRF [S1].

## How it works
**Inputs and Outputs:**
- Input: A pre-trained standard NeRF model (teacher) and a set of training images with camera poses.
- Output: A set of lightweight MLPs (students) arranged in a 3D grid, and a cached sparse 3D grid for surface localization.

**Data Structures:**
1.  **Student MLPs:** A collection of $N$ small shallow networks. Each network is significantly smaller than the original teacher NeRF.
2.  **Sparse 3D Grid:** A cached structure used to quickly locate the surface geometry, allowing the renderer to skip empty space [S3, unofficial].

**Training Procedure (Teacher-Student Distillation):**
1.  **Train Teacher:** Train a standard, deep NeRF on the scene data to convergence [S1].
2.  **Initialize Students:** Initialize a set of lightweight MLPs arranged in a 3D grid covering the scene volume [S1][S3, unofficial].
3.  **Distill Knowledge:** Distill the teacher NeRF's knowledge into the student MLPs. Each student MLP is responsible for capturing the radiance field information for a small portion of the space [S2, unofficial].
4.  **Optimize:** Optimize the student MLPs to match the teacher's output for samples drawn from the scene [S1].

**Rendering Procedure:**
1.  **Ray Casting:** For each pixel, cast a ray into the scene [S2, unofficial].
2.  **Surface Localization:** Use the cached sparse 3D grid to quickly locate the surface and determine the relevant region of interest, skipping empty space [S3, unofficial].
3.  **Dynamic Parameter Loading:** As the ray traverses the 3D grid, it enters different cells. Each cell has an associated small MLP. The renderer must dynamically load the parameters of the specific MLP corresponding to the current ray segment [S2, unofficial].
4.  **Volume Rendering:** Sample points along the ray. For each point, query the active small MLP to obtain density $\sigma$ and color $\mathbf{c}$. Perform standard volume rendering integration [S2, unofficial].

**Pitfalls and Edge Cases:**
-   **Memory Overhead:** Deploying a large number of MLPs incurs substantial memory overhead compared to a single shared network [S1].
-   **Dynamic Loading:** During rendering, a ray goes through different sets of NeRFs, each taking a different number of samples. This requires dynamic parameter loading, which necessitates a special CUDA kernel for proper speedup; naive implementation may negate performance gains [S2, unofficial].
-   **Precision:** Not given in the sources.

## Operating characteristics
-   **Time Complexity:** Acceleration of up to three orders of magnitude compared to the original NeRF [S1].
-   **Memory Complexity:** Substantial memory overhead due to the deployment of thousands of individual MLPs [S1].
-   **Strengths:** Significant speedup in rendering time; allows for real-time or near-real-time view synthesis by reducing per-point computation cost [S1][S3, unofficial].
-   **Limitations:** High memory usage; requires custom CUDA kernels for efficient dynamic parameter loading [S1][S2, unofficial].
-   **When to use:** When rendering speed is critical and memory budget allows for storing many small networks.
-   **When not to use:** When memory is severely constrained or when hardware does not support efficient dynamic parameter loading.

## Background
KiloNeRF was introduced by Reiser et al. in the paper "KiloNeRF: Speeding up Neural Radiance Fields with Thousands of Tiny MLPs" presented at ICCV 2021 [S1][S3, unofficial]. It addresses the computational bottleneck of standard NeRFs by decomposing the scene representation into thousands of tiny, specialized networks [S2, unofficial]. The method builds on the principle that a large deep NeRF is inefficient for representing small portions of space, as it processes global information for local queries [S2, unofficial].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
