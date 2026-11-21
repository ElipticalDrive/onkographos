# SZ-L/R
Also called: SZ with Lorenzo and linear regression predictors
## Overview
SZ-L/R is a block-based lossless/lossy compression algorithm for volumetric data that employs Lorenzo and linear regression predictors [S1]. It partitions input data into independent 6×6×6 blocks, allowing for cache-friendly processing and random access without inter-block dependencies [S1]. Unlike global interpolation methods such as [SZ-Interp](52-sz-interp.md), SZ-L/R prioritizes local feature preservation, making it suitable for unsmooth and irregular datasets, though it may introduce block-wise artifacts [S1].
## How it works
**Inputs:** A 3D volumetric dataset (e.g., AMR grid data).
**Outputs:** Compressed bitstream containing prediction errors and metadata.

**Data Structures:**
The input volume is partitioned into non-overlapping 6×6×6 blocks. Each block is processed independently.

**Steps:**
1.  **Partitioning:** Divide the entire input dataset into 6×6×6 blocks [S1].
2.  **Prediction Selection:** For each block, choose between the Lorenzo predictor or high-dimensional linear regression [S1]. The source does not specify the automatic selection criteria or if a fixed choice is made per block; it states "either... or" [S1].
3.  **Prediction:**
    *   **Lorenzo Predictor:** Predicts the current voxel value based on previously decoded neighbors within the block (typically using values from the previous layer and adjacent positions in the current layer) [S1].
    *   **Linear Regression:** Fits a high-dimensional linear regression model to the neighborhood of the current voxel within the block to predict its value [S1].
4.  **Error Calculation:** Compute the difference between the actual voxel value and the predicted value [S1].
5.  **Encoding:** Encode the prediction errors using arithmetic coding (inherited from the parent [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)) [S1].

**Parameters:**
*   Block size: 6×6×6 [S1].

**Edge Cases and Pitfalls:**
*   **Block-wise Artifacts:** The block-based nature can produce visible "block-wise" artifacts in the reconstructed data, particularly noticeable in smooth regions [S1].
*   **Boundary Handling:** Since blocks are independent, boundary effects between blocks are not smoothed, contributing to artifacts [S1].

## Subtopics

### SZ-Interp
SZ-Interp is a global compression algorithm that utilizes spline interpolation across all three dimensions of the entire dataset [S1]. Unlike SZ-L/R, it is not block-based, which leads to better rate-distortion performance (higher SSIM/PSNR) on smooth data like WarpX [S1]. However, it lacks the random access benefits and cache locality advantages of SZ-L/R, as it requires processing the entire dataset globally [S1].

## Operating characteristics
**Time and Memory Complexity:**
Block-based processing improves cache locality compared to global methods [S1]. Random access is supported because there is no data dependency between individual blocks, allowing partial visualization without decompressing the entire dataset [S1].

**Measured Performance:**
*   On smooth data (WarpX), SZ-Interp outperforms SZ-L/R in PSNR and SSIM [S1].
*   On unsmooth and irregular datasets (Nyx), SZ-L/R demonstrates better performance due to its focus on local features [S1].
*   Specific compression ratios (CR) and error metrics are provided in Table 2 of [S1] for various error bounds, but exact numerical values are not extracted in the notes.

**Strengths and Limitations:**
*   **Strengths:** Cache locality, random access support, partial visualization capability, better performance on irregular/unsmooth data [S1].
*   **Limitations:** Block-wise artifacts, potentially lower rate-distortion performance on smooth data compared to global methods [S1].

## Background
SZ-L/R is based on the SZ compression framework [S1]. It was evaluated in the context of AMR (Adaptive Mesh Refinement) applications developed by the AMReX framework, specifically Nyx cosmology simulation and WarpX electromagnetic simulations [S1]. The Lorenzo predictor is delineated in [13] and the linear regression approach in [S1] (referencing [13] for the combined method). The SZ-Interp variant utilizes spline interpolation as discussed in [34] [S1].

## Sources
[S1] Analyzing Impact of Data Reduction Techniques on Visualization for AMR Applications Using AMReX Framework, https://doi.org/10.1145/3624062.3625123, fetched 2026-10-27
