# Spatial Predictive Coding

Also called: Spatial Prediction for Feature Grids

## Overview
Spatial Predictive Coding reduces information redundancy in voxel grids by exploiting statistical dependence between spatially adjacent samples, analogous to adjacent pixel correlation in images [S1]. It sets itself apart from sibling methods by constructing a reference graph to select the optimal neighbor via Sum of Absolute Differences (SAD) and by performing quantization *before* prediction to improve parallelism [S1]. Unlike methods that predict density, this approach targets feature grids (e.g., color) where storage costs are significantly higher [S1].

## How it works
**Inputs and Outputs:**
The input is an original feature grid $x$ containing vectors such as color or NeRF features [S1]. The output is a bitstream containing reference indices and encoded residuals $y$ [S1].

**Data Structures:**
A Feature Grid is a 3D grid of vectors where each voxel $v$ holds a quantized feature vector $\hat{x}_v$ [S1]. A Reference Index is an integer identifier for each voxel indicating which adjacent voxel was selected as the predictor [S1]. A Residual is the difference between the current quantized feature and the reference quantized feature [S1].

**Steps:**
1. **Quantization:** Apply straightforward scalar quantization to the original feature vectors $x$ to obtain $\hat{x}$ [S1]. Vector quantization is avoided to prevent difficulties in subsequent prediction and bitrate estimation [S1].
2. **Reference Selection:** For each voxel, identify candidate neighbors using only complete voxels, excluding interpolated values at non-integer positions [S1]. Calculate the Sum of Absolute Differences (SAD) between the current voxel $\hat{x}$ and each candidate reference $\hat{x}_{ref}$:
   $$ \text{SAD}(\hat{x}, \hat{x}_{ref}) = \sum_{k} |\hat{x}_k - \hat{x}_{ref,k}| $$
   Select the reference $\hat{x}_{ref}$ that minimizes the SAD [S1]. Record the index of this reference for the current voxel [S1].
3. **Residual Calculation:** Compute the residual $y$ using the quantized values:
   $$ y = \hat{x} - \hat{x}_{ref} $$
   where $\hat{x}$ is the quantized feature to be coded and $\hat{x}_{ref}$ is the quantized reference feature [S1].
4. **Entropy Coding:** Encode the reference indices and the residuals $y$ into the bitstream using standard entropy coding [G]. The reference indices incur a high bit cost, which can be mitigated by partitioning voxels into blocks and selecting a single mode for all voxels within a block [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatial_predictive_coding(feature_grid):
    """
    feature_grid: 3D list of feature vectors (e.g., RGB or NeRF features)
    Returns: bitstream containing reference indices and residuals
    """
    # Step 1: Scalar Quantization [G]
    quantized_grid = [[round(v) for v in voxel] for voxel in feature_grid]
    
    bitstream = []
    processed = set()
    
    # Iterate in causal order (e.g., z, y, x)
    for z in range(len(quantized_grid)):
        for y in range(len(quantized_grid[0])):
            for x in range(len(quantized_grid[0][0])):
                current_val = quantized_grid[z][y][x]
                
                # Step 2: Reference Selection using SAD
                best_ref_val = None
                best_sad = float('inf')
                best_ref_index = None
                
                # Causal neighbors: only previously processed voxels
                for dz, dy, dx in [(-1, 0, 0), (0, -1, 0), (0, 0, -1)]:
                    nz, ny, nx = z + dz, y + dy, x + dx
                    if 0 <= nz < len(quantized_grid) and 0 <= ny < len(quantized_grid[0]) and 0 <= nx < len(quantized_grid[0][0]) and (nz, ny, nx) in processed:
                        ref_val = quantized_grid[nz][ny][nx]
                        sad = sum(abs(current_val[k] - ref_val[k]) for k in range(len(current_val)))
                        if sad < best_sad:
                            best_sad = sad
                            best_ref_val = ref_val
                            best_ref_index = (nz, ny, nx)
                
                # Step 3: Residual Calculation
                residual = [current_val[k] - best_ref_val[k] for k in range(len(current_val))]
                
                # Step 4: Encode [G]
                bitstream.append(best_ref_index)
                bitstream.append(residual)
                processed.add((z, y, x))
                
    return bitstream
```

**Parameters:**
*   **Reference Scope:** Only complete voxels are used as prediction values [S1]. Interpolated values at non-integer positions are excluded [S1].
*   **Target Data:** Prediction is performed only on the feature grid, not on density grids, because color feature grids typically cost much more storage [S1].
*   **Quantization Type:** Straightforward scalar quantization is used [S1].

**Edge Cases and Pitfalls:**
*   **High Bit Cost of Indices:** Selecting the optimal reference per voxel incurs a high bit cost for indexes [S1]. This is a significant pitfall [S1]. Mitigation involves partitioning voxels into blocks and selecting a single mode for all voxels within a block [S1].
*   **Quantization Order:** Quantizing before prediction (rather than after) is critical for parallelism and ensuring the residual calculation uses consistent quantized values [S1].

## Operating characteristics
*   **Complexity:** The reference selection step requires computing SAD for all neighbors for each voxel, leading to higher computational complexity compared to fixed predictors [S1]. The bit cost of reference indices is a major factor in compression efficiency [S1].
*   **Performance:** The method is effective for feature grids where strong spatial correlation exists [S1]. It achieves good compression ratios for color/feature data but may be less efficient for density grids due to the overhead of index coding [S1].
*   **Strengths:** The method exploits strong spatial redundancy [S1]. It is parallelizable due to pre-quantization [S1]. Simple scalar quantization is efficient enough as residuals are often near zero [S1].
*   **Limitations:** The method incurs a high bit cost for reference indices if not block-quantized [S1]. It is limited to complete voxels with no interpolation [S1]. Scalar quantization may be suboptimal compared to vector quantization if prediction and bitrate estimation were handled differently [S1].

## Background
Spatial Predictive Coding in the context of voxel-based radiance fields was proposed in "SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field" [S1]. The method addresses the high storage costs of feature grids in NeRF compression by treating feature voxels similarly to image pixels, leveraging adjacent correlation [S1]. It integrates with rate-distortion joint finetuning to optimize compression performance across different bitrates [S1].

## Sources
[S1] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
