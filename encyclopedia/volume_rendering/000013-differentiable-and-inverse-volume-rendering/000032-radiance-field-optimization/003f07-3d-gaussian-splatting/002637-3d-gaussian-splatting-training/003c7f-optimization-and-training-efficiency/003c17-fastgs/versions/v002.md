# FastGS

Also called: Fast Gradient Checkpointing and Dynamic Sparse Rasterization

## Overview
FastGS is a method that re-engineers the 3D Gaussian Splatting training loop to accelerate optimization by combining dynamic sparse rasterization with fast gradient checkpointing [S1, unofficial]. It sets itself apart from standard training by skipping tiles with negligible transmittance during rasterization and recomputing only necessary intermediate values during the backward pass, rather than storing all activations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians (position, covariance, opacity, color, SH coefficients), camera parameters, and a target image.
*   **Output:** Optimized Gaussian parameters and rendered images.

**Data Structures:**
The method relies on the standard 3D Gaussian representation but optimizes the memory access patterns during rasterization. Specifically, it utilizes a rearranged splat data structure to fit 128-bit aligned vectors for efficient memory loading [S3].
*   **Color:** Padded 4D float vector (`vec4`), 128 bits. (Original was 96-bit `vec3`, which lacked optimized load instructions) [S3].
*   **Bounding Box:** 4D unsigned integer vector (`uvec4`) in pixel space using 16-bit integers, 64 bits [S3].
*   **Screen Position:** 2D float vector (`vec2`), 64 bits [S3].
*   **Packing:** The Bounding Box and Screen Position are packed together to form a single 128-bit structure alongside the Color vector [S3].

**Steps:**
1.  **Dynamic Sparse Rasterization:**
    *   Divide the image into tiles.
    *   For each tile, calculate the transmittance.
    *   Skip rasterization for tiles where transmittance is negligible (i.e., the tile is fully opaque or empty) [S1, unofficial].
2.  **Forward Pass:**
    *   Render the image by alpha-blending Gaussians sorted by depth within active tiles.
    *   Store intermediate values required for gradient computation.
3.  **Loss Calculation:**
    *   Compute the loss $\mathcal{L}$ (e.g., L1, SSIM) between the rendered image and the target image.
4.  **Fast Gradient Checkpointing (Backward Pass):**
    *   Instead of storing all intermediate activations from the forward pass, store only a subset of "checkpoints."
    *   During backpropagation, recompute the necessary intermediate values from the checkpoints to calculate gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ [S1, unofficial].
    *   This reduces memory usage, allowing for larger batch sizes or more complex scenes, while trading compute time for memory [S1, unofficial].

**Parameters:**
*   **Threshold for Negligible Transmittance:** Not given in the sources.
*   **Checkpoint Interval:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Memory vs. Compute Trade-off:** Gradient checkpointing reduces memory but increases computation time. If the checkpoint interval is too large, recomputation overhead may negate memory benefits [S1, unofficial].
*   **Tile Culling Accuracy:** Skipping tiles based on transmittance requires accurate bounds. If bounds are too loose, unnecessary computation occurs; if too tight, artifacts may appear [S1, unofficial].
*   **Data Structure Alignment:** Using the original 96-bit color vector leads to inefficient memory access. The 128-bit padding is critical for performance [S3].

## Subtopics

### Accelerated Training for 3DGS
A strategy to improve training time efficiency by separating higher SH bands from the 0th band within the rasterizer, reducing the number of updates for higher SH bands compared to diffused colors [S2]. It is chosen when training time is the primary bottleneck and view-dependent effects are less critical than diffuse color [S2].
*   **Steps:**
    1.  Modify the rasterizer to split SH bands from the diffused color [S2].
    2.  Update SH bands every 16 iterations, while updating diffused color at every step [S2].
    3.  Optimize SSIM loss calculation by replacing the standard 11x11 2D Gaussian kernel with two smaller 1D Gaussian kernels, calculated via a fused CUDA kernel [S2].
*   **Formula:** The total loss $L$ is defined as:
    $$L = (1 - \lambda_{SSIM})L_1 + \lambda_{SSIM} L_{D-SIM} + \lambda_m L_m + \lambda_{sh} L_{sh}$$
    where $L_m$ and $L_{sh}$ are defined in papers [11] and [13] respectively [S2].

### 128-bit Splat Data Structure
Restructuring splat attributes (color, bounding box, position) into 128-bit aligned vectors to utilize optimized memory load instructions and reduce the total number of memory operations [S3]. It is chosen to maximize GPU memory bandwidth utilization [S3].
*   **Layout:**
    *   **Field 1: Color.** Type: 4D float vector (`vec4`). Size: 128 bits. Note: Original was 96-bit 3D float (`vec3`), padded to 4D [S3].
    *   **Field 2: Bounding Box.** Type: 4D unsigned integer vector (`uvec4`). Size: 64 bits. Note: Stored in pixel space using 16-bit unsigned integers [S3].
    *   **Field 3: Screen Position.** Type: 2D float vector (`vec2`). Size: 64 bits. Packing: Bounding Box (64-bit) is packed alongside Screen Position (64-bit) to form a single 128-bit structure [S3].
*   **Pitfall:** Using the original 96-bit 3D float vector for color does not have an optimized load instruction, leading to inefficient memory access [S3].
*   **Comparison:** The rearrangement trades additional compute instructions required to unpack the data for further reductions in memory operation instructions [S3].

## Operating characteristics
*   **Time Complexity:** Reduced training time due to skipping negligible tiles and less frequent SH updates (in Accelerated Training variant) [S1, unofficial][S2].
*   **Memory Complexity:** Reduced memory usage during backpropagation due to gradient checkpointing [S1, unofficial].
*   **Measured Performance:** MMGS, which builds on these principles, achieves 10x accelerated training speed with only 10% primitives [S1, unofficial].
*   **Strengths:** Faster convergence, lower memory footprint, efficient use of GPU memory bandwidth [S1, unofficial][S3].
*   **Limitations:** Increased compute overhead for gradient recomputation [S1, unofficial]. Potential artifacts if tile culling is inaccurate [S1, unofficial].
*   **When to Use:** When training time is a bottleneck and GPU memory is limited [S1, unofficial][S2].
*   **When Not to Use:** When real-time inference speed is more critical than training time, or when view-dependent effects are paramount and frequent SH updates are required [S2].

## Background
FastGS was introduced to address the computational inefficiencies of standard 3D Gaussian Splatting training [S1, unofficial]. It re-engineers the training loop by integrating dynamic sparse rasterization and fast gradient checkpointing [S1, unofficial]. The method is part of a broader effort to optimize 3D Gaussian Splatting, including variants like MMGS and Trick-GS [S1, unofficial][S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S3] Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting, https://doi.org/10.1145/3820019, fetched 2026-11-01
