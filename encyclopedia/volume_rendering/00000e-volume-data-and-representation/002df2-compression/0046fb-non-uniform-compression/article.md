# Non-uniform Compression
Also called: Importance-based Nonuniform Compression

## Overview
Non-uniform Compression (NCB) is a compression strategy that applies different downsampling scales to density and color features to balance storage cost and rendering quality [S1]. It sets itself apart from uniform compression methods by recognizing that density contributes more to preserving visual quality while color features occupy significantly more storage, justifying asymmetric treatment [S1]. It builds on [Sparse structures](5-sparse-structures.md) by integrating sparsification with non-uniform scaling.

## How it works
**Inputs and Outputs:**
*   **Input:** Dense voxel grids for volume density ($V_\sigma$) and volume color ($V_c$).
*   **Output:** Compressed bitstream containing downsampled sparse grids, a full-resolution occupancy mask, and metadata.

**Data Structures:**
1.  **Downsampled Grids:** Sparse voxel grids for density and color at reduced resolutions.
2.  **Occupancy Mask:** A binary mask at the original full resolution indicating occupied voxels.
3.  **Data Pointers:** Reversible mapping from the binary mask to sparse array indices.

**Parameters:**
*   **Density downsampling scale:** $1/2$ by default [S1].
*   **Color downsampling scale:** $1/4$ by default [S1].
*   **Data pointer storage:** 4 bytes reduced to 1 bit per voxel [S1].
*   **Precision:** Float 16 (FP16) [S1].

**Steps:**
1.  **Downsampling:** Treat volume density ($V_\sigma$) and volume color ($V_c$) as dense voxel grids and directly apply trilinear downsampling using their respective scales [S1].
2.  **Sparsification:** Follow the sparsification operation in the Plenoxels pipeline to re-sparsify the downsampled voxel grids [S1].
3.  **Mask Generation:** Keep the full-resolution occupancy mask for future decompression. This mask indicates if a voxel is occupied [S1].
4.  **Pointer Conversion:** Convert data pointers (used in importance map and sparse array storage) to the binary occupancy mask. This conversion is reversible to derive data pointers from the mask [S1].
5.  **Encoding:** Convert all tensors to float 16 and compress with zlib for saving [S1].

**Decompression Steps:**
1.  **Decoding:** Decompress the bitstream to retrieve the downsampled sparse grids and the full-resolution binary mask [S1].
2.  **Upsampling:** Trilinearly upsample the downsampled density ($V_\sigma$) and color ($V_c$) grids to get the full-size voxel grid [S1].
3.  **Sparsification:** Sparsify the upsampled grid using the saved full-resolution mask [S1].
4.  **Pointer Derivation:** Derive data pointers from the mask if required for rendering [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_volume(V_sigma, V_color, density_scale=0.5, color_scale=0.25):
    # Step 1: Downsample
    V_sigma_down = trilinear_downsample(V_sigma, density_scale)
    V_color_down = trilinear_downsample(V_color, color_scale)
    
    # Step 2: Sparsify downsampled grids (using Plenoxels pipeline logic)
    # Assume get_occupancy_mask returns a binary mask of occupied voxels
    # Note: The source implies sparsification happens on the downsampled grid
    # and the mask is kept at full resolution for decompression.
    # However, the mask generation step says "keep the full-resolution mask".
    # This implies the mask is derived from the original or upsampled context?
    # The source says: "sparsify the upsampled grid with the saved full-resolution mask"
    # during decompression. During compression, we need to generate this mask.
    # The source states: "Keep the full-resolution mask for future decompression."
    # It does not explicitly state how the full-resolution mask is generated 
    # from the downsampled data, but implies it's part of the pipeline.
    # We assume the mask represents the occupancy of the original volume 
    # or the importance-based selection.
    # Given "Importance-based" alias, we assume an importance metric selects voxels.
    # For implementation completeness based on "re-sparsify... and keep mask":
    # We assume the mask is generated from the original data or an importance map.
    # Since the source doesn't give the importance metric, we note this gap.
    # [G] The source mentions "importance map" but doesn't define the metric.
    # We assume the mask is pre-computed or derived from an importance metric.
    occupancy_mask = generate_full_resolution_occupancy_mask(V_sigma) # [G] Placeholder for importance logic
    
    # Step 3: Convert pointers to mask (reversible)
    # The source says data pointers are converted to a binary mask.
    # This implies the sparse storage uses pointers, which are then converted to a mask for storage?
    # Or the mask is stored separately.
    # "The data pointers ... are converted to a binary mask ... and the conversation is reversible"
    # This suggests the mask is the primary storage format for occupancy, replacing pointers.
    
    # Step 4: Encode
    V_sigma_down_fp16 = to_float16(V_sigma_down)
    V_color_down_fp16 = to_float16(V_color_down)
    occupancy_mask_binary = to_binary(occupancy_mask)
    
    compressed_data = zlib_compress([V_sigma_down_fp16, V_color_down_fp16, occupancy_mask_binary])
    return compressed_data

def decompress_volume(compressed_data):
    # Step 1: Decode
    V_sigma_down_fp16, V_color_down_fp16, occupancy_mask_binary = zlib_decompress(compressed_data)
    
    # Step 2: Upsample
    V_sigma_up = trilinear_upsample(V_sigma_down_fp16, 1.0/density_scale)
    V_color_up = trilinear_upsample(V_color_down_fp16, 1.0/color_scale)
    
    # Step 3: Sparsify with mask
    V_sigma_sparse = sparsify(V_sigma_up, occupancy_mask_binary)
    V_color_sparse = sparsify(V_color_up, occupancy_mask_binary)
    
    return V_sigma_sparse, V_color_sparse
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using FP16 may introduce quantization error, but the source reports it as the standard for storage [S1].
*   **Mask Accuracy:** The quality of the full-resolution mask is critical. If the mask does not accurately reflect important voxels, visual artifacts may occur [S1].
*   **Reversibility:** The conversion between data pointers and the binary mask must be strictly reversible to ensure correct reconstruction [S1].

## Operating characteristics
*   **Time Complexity:** Decompression involves trilinear upsampling and sparsification, which are linear with respect to the number of voxels in the upsampled grid [S1].
*   **Memory Complexity:** Storage is reduced by downsampling color features more aggressively than density [S1].
*   **Measured Performance:** On the LLFF dataset, saving 2.5% of the most important voxels adds 2.6 MB to the size, increasing PSNR from 31.02 to 31.35 dB. Saving 10% adds 10.0 MB, increasing PSNR to 31.54 dB [S1].
*   **Strengths:** Significant storage reduction (over 40x compared to direct Plenoxels) with minimal quality loss (0.26 dB) [S1].
*   **Limitations:** Requires training a Neural Codebook (NCB) for each scene, which adds deployment time [S1].
*   **When to use:** When storage size is a critical constraint and per-scene training is acceptable.
*   **When not to use:** When real-time decompression without prior training is required.

## Background
Non-uniform Compression was introduced in the paper "Compact Real-Time Radiance Fields with Neural Codebook" [S1]. It addresses the high storage costs of radiance field representations by leveraging the observation that density and color features have different impacts on visual quality and storage requirements.

## Sources
[S1] Compact Real-Time Radiance Fields with Neural Codebook, https://doi.org/10.1109/icme55011.2023.00374, fetched 2026-10-26
