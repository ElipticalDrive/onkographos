# Feature Prediction Table Encoding

Also called: Context-based voxel prediction encoding, Multi-level feature prediction

## Overview
Feature Prediction Table Encoding is a lossless compression method for voxelized models that reduces redundancy by predicting voxel occupancy based on local context, storing only the binary residual between the prediction and the true value [S1]. It differs from existing Sparse Voxel Octree (SVO) approaches by modeling prediction contexts at full grid resolution with varying strides, rather than using local contexts of SVO nodes with different resolutions at each level [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary voxel grid of dimensions $X \times Y \times Z$.
*   **Output:** A binary stream containing a header (levels $p$, feature prediction tables) and a PSVO-encoded residual grid [S1].

**Data Structures:**
*   **Feature Prediction Tables (FPT):** Three tables concatenated, totaling 48 bytes per level [S1].
    *   Table 1: 256 bits ($2^8$), indexed by 8 context bits [S1].
    *   Table 2: 64 bits ($2^6$), indexed by 6 context bits [S1].
    *   Table 3: 64 bits ($2^6$), indexed by 6 context bits [S1].
*   **Header:**
    *   1 byte: Number of prediction levels $p$. If $p=0$, a single table (from the top level) is used for all levels [S1].
    *   $p \times 48$ bytes: The FPTs [S1].
*   **Residual Grid:** A grid of the same dimensions as the input, storing actual occupancy values for "key voxels" and binary residuals for predicted voxels [S1].

**Parameters:**
*   $p$: Number of prediction levels. Determines the initial stride $s_0 = 2^p$ [S1]. Typical value: 3 for synthetic tree datasets [S1].
*   $s_i$: Stride at level $L_i$, calculated as $s_i = 2^{p-i}$ [S1].
*   $r_i$: Half the stride at level $L_i$, calculated as $r_i = s_i / 2$ [S1].

**Algorithm Steps:**

1.  **Construct Feature Prediction Tables:**
    For each prediction step $k \in \{1, 2, 3\}$, analyze the voxel grid. For every combination of context voxel occupancies, determine the most frequent occupancy value of the corresponding target voxel across the whole grid. Store these values in the respective FPT [S1].

2.  **Initialize Level $L_0$:**
    Identify key voxels at coordinates $(q_x \cdot s_0, q_y \cdot s_0, q_z \cdot s_0)$ for $q_x \in \{0, \dots, \lfloor X/s_0 \rfloor\}$, $q_y \in \{0, \dots, \lfloor Y/s_0 \rfloor\}$, $q_z \in \{0, \dots, \lfloor Z/s_0 \rfloor\}$ [S1]. Store their actual occupancy values directly in the residual grid [S1].

3.  **Execute Three Prediction Steps per Level:**
    For each level $L_i$ (from $i=0$ to $p-1$), with stride $s_i$ and offset $r_i = s_i/2$:

    *   **Step 1 (Central Voxels):**
        Predict the occupancy of central voxels $v_{x,y,z}$ using the nearest eight key voxels $v_{x \pm r_i, y \pm r_i, z \pm r_i}$ [S1].
        *   Context: 8 bits from key voxels.
        *   Prediction: Lookup in FPT 1.
        *   Residual: $r = \text{true} \oplus \text{prediction}$ [S1].
        *   Store $r$ in the residual grid at $(x,y,z)$ [S1].

    *   **Step 2 (Side Centers):**
        Predict the occupancy of cube face centers. For a face orthogonal to the x-axis, the target is $v_{x,y,z}$. The context consists of four surrounding key voxels $v_{x, y \pm r_i, z \pm r_i}$ and two nearest voxels determined in Step 1, $v_{x \pm r_i, y, z}$ [S1].
        *   Context: 6 bits (4 from keys, 2 from Step 1 residuals).
        *   Prediction: Lookup in FPT 2.
        *   Residual: $r = \text{true} \oplus \text{prediction}$ [S1].
        *   Store $r$ in the residual grid [S1].

    *   **Step 3 (Edge Centers):**
        Predict the occupancy of cube edge centers. For an edge parallel to the z-axis, the target is $v_{x,y,z}$. The context consists of two nearest corner voxels $v_{x,y, z \pm r_i}$ and four nearest face centers from Step 2, $v_{x \pm r_i, y \pm r_i, z}$ [S1].
        *   Context: 6 bits (2 from keys, 4 from Step 2 residuals).
        *   Prediction: Lookup in FPT 3.
        *   Residual: $r = \text{true} \oplus \text{prediction}$ [S1].
        *   Store $r$ in the residual grid [S1].

    *Note: The residuals from previous steps serve as the context for subsequent steps within the same level [S1].*

4.  **Recurse to Next Level:**
    Once level $L_i$ is finished, the encoded voxels (key voxels + residuals) become the key voxels for level $L_{i+1}$ [S1]. Repeat steps 2-3 with stride $s_{i+1} = s_i / 2$ [S1].

5.  **Post-processing (PSVO Encoding):**
    The resulting residual grid is encoded using Predictive Sparse Voxel Octree (PSVO) [S1]. The PSVO is linearized by concatenating codes in depth-first traversal order of octree nodes [S1].

6.  **Decoding:**
    *   Read $p$ and FPTs from the header [S1].
    *   Reconstruct the mixed grid of key voxel values and residuals from the PSVO code [S1].
    *   Perform the same coarse-to-fine prediction procedure [S1].
    *   For each voxel, combine the prediction from the FPT with the stored residual to infer the true value: $\text{true} = \text{prediction} \oplus \text{residual}$ [S1].
    *   Store the inferred value in the corresponding grid cell [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** For non-cubic grids or grids with dimensions not equal to $2^d$, prediction contexts may include non-existing voxels outside the grid. These out-of-bounds voxels are treated as having an occupancy value of 0 [S1].

## Operating characteristics
*   **Compression Performance:** On a synthetic tree dataset (average occupancy 0.277%, 63.4M voxels), prediction-based encoding reduced dataset size by 15.6% relative to basic PSVO and 13.7% relative to PSVO+CBO [S1].
*   **Comparison with CBO:** Combining prediction with Complete Branch Optimization (CBO) was not beneficial; accurate predictions generate zero residuals, which undermines CBO efficiency [S1]. PSVO+CBO alone achieved only a 1.9% decrease over basic PSVO [S1].
*   **Theoretical Limits:** The entropy of the synthetic tree dataset was 0.02839 bits/voxel, with a theoretical lower bound of 85.823 MiB for the raw 2.953 GiB dataset [S1].
*   **LiDAR Data:** On a LiDAR tree dataset (average occupancy 0.558%, 17.5M voxels), the entropy was 0.04122 bits/voxel, with a lower bound of 59.4 MiB for the raw 1.407 GiB dataset [S1].
*   **Strengths:** Effective for models with high structural redundancy and low occupancy, such as tree models [S1].
*   **Limitations:** Requires statistical analysis of the model to build FPTs, which adds preprocessing overhead. Not beneficial when combined with CBO [S1].

## Background
The method was proposed for encoding voxelized tree models for machine learning-based applications [S1]. It was introduced in a 2025 paper by authors associated with IEEE Access [S1]. The approach distinguishes itself from traditional SVO methods by maintaining full grid resolution for prediction contexts while varying the stride (spacing) between context voxels across levels [S1].

## Sources
[S1] Efficient Encoding and Decoding of Voxelized Models for Machine Learning-Based Applications, https://doi.org/10.1109/access.2025.3526202, fetched 2026-10-27
