# Spatial Predictive Coding

Also called: Spatial Prediction for Feature Grids

## Overview
Spatial Predictive Coding reduces information redundancy in voxel grids by exploiting statistical dependence between spatially adjacent samples, analogous to adjacent pixel correlation in images [S1]. It sets itself apart from sibling methods by constructing a reference graph to select the optimal neighbor via Sum of Absolute Differences (SAD) and by performing quantization *before* prediction to improve parallelism [S1]. Unlike methods that predict density, this approach targets feature grids (e.g., color) where storage costs are significantly higher [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A quantized feature grid $\hat{X}$ (typically color/feature vectors, not density) [S1].
*   **Output:** A bitstream containing reference indices and encoded residuals [S1].

**Data Structures:**
*   **Feature Grid:** A 3D grid of vectors. Each voxel $v$ holds a quantized feature vector $\hat{x}_v$.
*   **Reference Index:** An integer identifier for each voxel, indicating which adjacent voxel was selected as the predictor.
*   **Residual:** The difference between the current quantized feature and the reference quantized feature.

**Steps:**
1.  **Quantization:** Apply straightforward scalar quantization to the original feature vectors $x$ to obtain $\hat{x}$. Vector quantization is avoided to prevent difficulties in subsequent prediction and bitrate estimation [S1].
2.  **Reference Selection:** For each voxel, identify candidate neighbors (complete voxels only; interpolated values at non-integer positions are not used) [S1]. Calculate the Sum of Absolute Differences (SAD) between the current voxel $\hat{x}$ and each candidate reference $\hat{x}_{ref}$:
    $$ \text{SAD}(\hat{x}, \hat{x}_{ref}) = \sum_{k} |\hat{x}_k - \hat{x}_{ref,k}| $$
    Select the reference $\hat{x}_{ref}$ that minimizes the SAD. Record the index of this reference for the current voxel [S1].
3.  **Residual Calculation:** Compute the residual $y$ using the quantized values:
    $$ y = \hat{x} - \hat{x}_{ref} $$
    where $\hat{x}$ is the quantized feature to be coded and $\hat{x}_{ref}$ is the quantized reference feature [S1].
4.  **Entropy Coding:** Encode the reference indices and the residuals $y$ into the bitstream. The reference indices incur a high bit cost, which can be mitigated by partitioning voxels into blocks and selecting a single mode for all voxels within a block [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatial_predictive_coding(feature_grid):
    """
    feature_grid: 3D array of feature vectors (e.g., RGB or NeRF features)
    Returns: bitstream containing reference indices and residuals
    """
    # Step 1: Scalar Quantization
    # Apply scalar quantization to each component of the feature vectors
    quantized_grid = scalar_quantize(feature_grid)
    
    bitstream = []
    
    # Optional: Block partitioning to reduce index cost
    # If block mode is enabled, group voxels into blocks and select one reference mode per block
    
    for voxel in quantized_grid:
        current_val = voxel.value
        
        # Step 2: Reference Selection using SAD
        best_ref_val = None
        best_sad = infinity
        best_ref_index = None
        
        # Iterate over causal neighbors (complete voxels only)
        for neighbor in get_causal_neighbors(voxel):
            ref_val = neighbor.value
            # Calculate Sum of Absolute Differences
            sad = sum(abs(current_val[k] - ref_val[k]) for k in range(len(current_val)))
            
            if sad < best_sad:
                best_sad = sad
                best_ref_val = ref_val
                best_ref_index = neighbor.index
                
        # Step 3: Residual Calculation
        # y = x_hat - x_hat_ref
        residual = current_val - best_ref_val
        
        # Step 4: Encode
        bitstream.append(encode_index(best_ref_index))
        bitstream.append(encode_residual(residual))
        
    return bitstream
```

**Parameters:**
*   **Reference Scope:** Only complete voxels are used as prediction values. Interpolated values at non-integer positions are excluded [S1].
*   **Target Data:** Prediction is performed only on the feature grid, not on density grids, because color feature grids typically cost much more storage [S1].
*   **Quantization Type:** Straightforward scalar quantization is used [S1].

**Edge Cases and Pitfalls:**
*   **High Bit Cost of Indices:** Selecting the optimal reference per voxel incurs a high bit cost for indexes. This is a significant pitfall. Mitigation involves partitioning voxels into blocks and selecting a single mode for all voxels within a block [S1].
*   **Quantization Order:** Quantizing before prediction (rather than after) is critical for parallelism and ensuring the residual calculation uses consistent quantized values [S1].

## Operating characteristics
*   **Complexity:** The reference selection step requires computing SAD for all neighbors for each voxel, leading to higher computational complexity compared to fixed predictors. The bit cost of reference indices is a major factor in compression efficiency.
*   **Performance:** Effective for feature grids where strong spatial correlation exists. The method achieves good compression ratios for color/feature data but may be less efficient for density grids due to the overhead of index coding [S1].
*   **Strengths:** Exploits strong spatial redundancy; parallelizable due to pre-quantization; simple scalar quantization is efficient enough as residuals are often near zero [S1].
*   **Limitations:** High bit cost for reference indices if not block-quantized; limited to complete voxels (no interpolation); scalar quantization may be suboptimal compared to vector quantization if prediction/bitrate estimation were handled differently [S1].

## Background
Spatial Predictive Coding in the context of voxel-based radiance fields was proposed in "SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field" [S1]. The method addresses the high storage costs of feature grids in NeRF compression by treating feature voxels similarly to image pixels, leveraging adjacent correlation. It integrates with rate-distortion joint finetuning to optimize compression performance across different bitrates [S1].

## Sources
[S1] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
