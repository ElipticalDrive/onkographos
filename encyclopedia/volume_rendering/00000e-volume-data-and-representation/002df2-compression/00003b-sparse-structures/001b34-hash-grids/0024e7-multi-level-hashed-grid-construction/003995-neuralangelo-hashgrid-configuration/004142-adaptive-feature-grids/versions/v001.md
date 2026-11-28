# Adaptive Feature Grids

Also called: APMGSRN

## Overview
Adaptive Feature Grids (APMGSRN) is a neural volume compression method that uses multiple spatially adaptive feature grids with learnable transformation matrices to concentrate representational capacity in high-error regions, addressing the inefficiency of fixed uniform placement [S1, unofficial]. It sets itself apart from its parent, Neuralangelo Hashgrid Configuration, by replacing static hash-based spatial indexing with learned, adaptive grid transformations that dynamically allocate parameters based on feature density, though it underperforms hash encodings on highly sparse data where hash collision averaging is beneficial [S1, unofficial].

## How it works
**Inputs:** A 3D volume dataset (potentially exceeding GPU memory); a target compression ratio or error threshold.
**Outputs:** A set of learnable transformation matrices and feature grid weights; a compressed model file.

**Data Structures:**
*   **Adaptive Grids:** A collection of $N$ grids, where $N \in [16, 32]$. Each grid consists of a learnable transformation matrix $T_i$ and a feature tensor $F_i$.
*   **Bricks:** For large volumes, the domain is decomposed into bricks. Each brick is processed independently but shares the global grid parameters.
*   **Ghost Cells:** Boundary cells added to each brick to mitigate artifacts during parallel processing.

**Steps:**
1.  **Initialization:** Initialize $N$ adaptive grids with learnable transformation matrices $T_i$ and feature tensors $F_i$. The number of grids $N$ is typically set between 16 and 32 [S1, unofficial].
2.  **Domain Decomposition:** If the volume exceeds GPU memory, split the data into bricks. Assign bricks to parallel GPU processes. Add ghost cells to each brick to handle boundary continuity [S1, unofficial].
3.  **Feature Density Loss Calculation:** Compute a feature density loss that guides the training. This loss function is designed to concentrate capacity in regions with high reconstruction error. The exact formula for the feature density loss is not provided in the sources [G: The loss likely penalizes low feature density in high-error regions, encouraging the transformation matrices to map high-error voxel coordinates to dense feature regions].
4.  **Training:** Train the $N$ adaptive grids using the feature density loss. The learnable transformation matrices $T_i$ adjust to map input coordinates to optimal feature locations. Training is performed per brick in parallel across GPUs [S1, unofficial].
5.  **Compression:** After training, the model weights (transformation matrices and feature tensors) are serialized. For the improved variant, apply SZ3 post-compression to the regularized feature grids [S1, unofficial].

**Parameters:**
*   **Number of Adaptive Grids ($N$):** 16–32 [S1, unofficial].
*   **Brick Size:** Not given in the sources.
*   **Ghost Cell Width:** Not given in the sources.

**Pitfalls:**
*   **Sparse Data:** Underperforms hash encodings on highly sparse data (e.g., asteroid datasets) where hash collision averaging provides beneficial regularization [S1, unofficial].
*   **Memory:** Requires significant GPU memory for the initial training of the full set of adaptive grids if not using domain decomposition [S1, unofficial].

## Subtopics

### Improved APMGSRN
An improved version of APMGSRN that reduces model size by 80% through custom CUDA kernels and SZ3 post-compression of regularized feature grids, with less than 0.1 dB loss [S1, unofficial]. It is chosen when extreme compression ratios are needed for sparse or complex datasets like asteroid volumes, achieving 49.04 dB PSNR at 1,529× compression on a 111.71 GB dataset with a 77 MB model [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time scales with the number of bricks and GPUs. Average training time is 35 seconds per brick on 8 GPUs for the Channel flow volume [S1, unofficial].
*   **Memory Complexity:** Model size is significantly smaller than the original volume. For the 4096³ Rotstrat volume (250 GB), the model size is 55 MB [S1, unofficial]. For the Channel flow volume (450 GB), the model size is 97 MB [S1, unofficial].
*   **Measured Performance:**
    *   **Rotstrat Volume (4096³, 250 GB):** 41.32 dB PSNR at 4791× compression with a 55 MB model [S1, unofficial].
    *   **Channel Flow Volume (7680 × 1568 × 10240, 450 GB):** 40.06 dB PSNR with a 97 MB model [S1, unofficial].
    *   **Asteroid Dataset (111.71 GB, 100 timesteps):** 49.04 dB PSNR at 1,529× compression with a 77 MB model (Improved APMGSRN) [S1, unofficial].
*   **Strengths:** High compression ratios for dense volumes; parallelizable via domain decomposition; adaptive capacity allocation reduces waste in homogeneous regions.
*   **Limitations:** Poor performance on highly sparse data compared to hash encodings; requires careful tuning of the feature density loss.
*   **When to Use:** Dense scientific volumes where high compression is needed and data is not highly sparse.
*   **When Not to Use:** Highly sparse data where hash collision averaging is beneficial.

## Background
Adaptive Feature Grids (APMGSRN) was introduced to address the inefficiency of fixed uniform placement in feature grids for neural volume rendering [S1, unofficial]. It builds on the concept of spatially adaptive grids, using learnable transformation matrices to concentrate capacity in high-error regions [S1, unofficial]. The method was validated at HPC scale, demonstrating its ability to handle tera-scale data on consumer hardware through domain decomposition and parallel processing [S1, unofficial]. An improved version of APMGSRN was later developed to further reduce model size using custom CUDA kernels and SZ3 post-compression [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
