# SPC-NeRF

Also called: Spatial Predictive Compression for Voxel-Based Radiance Field

## Overview
SPC-NeRF is a Rate-Distortion (RD) optimized predictive coding framework designed to compress Explicit Voxel Grid (EVG) represented Neural Radiance Fields (NeRFs) by removing spatial redundancy [S1]. It distinguishes itself from sibling methods like DirectVoxGO by introducing a two-stage finetuning process and a reference graph that jointly optimizes compression ratio and rendering distortion during training, achieving 32% bit savings compared to the state-of-the-art VQRF method [S1].

## How it works
**Inputs:** An uncompressed voxel grid obtained from the DirectVoxGO pipeline [S1].
**Outputs:** A compressed bitstream containing reference indexes, residuals, masks, and model weights [S1].

**Data Structures:**
The method operates on an EVG where each voxel stores density and color features. The compressed bitstream syntax elements are:
1.  Reference indexes: Index pointing to the chosen adjacent voxel for prediction.
2.  Coarse residuals: Quantized differences between actual and predicted values for non-critical voxels.
3.  Fine residuals: Quantized differences for critical voxels, reconstructed after post-finetuning.
4.  Prune mask: Binary indicator for pruned vs. unpruned voxels.
5.  Critical mask: Binary indicator for critical vs. non-critical voxels among unpruned ones.
6.  Non-prune density grid: Density values for unpruned voxels.
7.  Weights of the shallow MLP: Model parameters.
8.  Metadata: Scene-specific information [S1].

**Algorithm Steps:**

1.  **Initialization:** Apply the DirectVoxGO pipeline to obtain the initial uncompressed voxel grid [S1].
2.  **Pruning and Critical Selection:** Prune approximately 90% of unimportant voxels based on a pruning ratio parameter. Mark a small subset of the remaining unpruned voxels as "critical" [S1].
3.  **Reference Graph Construction:** For each unpruned voxel, select one adjacent voxel from seven possible previously encountered neighboring voxels as the prediction value. Record the index of this reference to construct a reference relation graph [S1].
4.  **RD Loss Formulation and Finetuning:** Apply a unique bit estimation method to estimate the bitrate of the residuals. Formulate a Rate-Distortion loss function combining rendering distortion and estimated bitrate. Finetune the voxel grid to minimize this loss [S1].
5.  **Coarse Residual Generation:** Perform actual quantization and prediction using the reference graph to generate coarse residuals for non-critical voxels [S1].
6.  **Post-Finetuning:** Freeze all non-critical voxels. Perform a second finetuning step to generate fine residuals specifically for reconstructing the critical voxels [S1].
7.  **Entropy Coding:**
    *   Encode the reference indexes, coarse residuals, and fine residuals using arithmetic coding [S1].
    *   Encode the prune mask, critical mask, non-prune density grid, shallow MLP weights, and metadata using a simple LZ77 codec, as these occupy little storage relative to the residuals [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spc_nerf_compress(uncompressed_voxel_grid):
    """
    Compress an uncompressed voxel grid using SPC-NeRF.
    """
    # Step 1: Pruning and Critical Selection
    # Prune ~90% of voxels. Mark some remaining as critical.
    prune_mask, critical_mask = prune_and_select_critical(uncompressed_voxel_grid, pruning_ratio=0.9)
    unpruned_voxels = get_unpruned_voxels(uncompressed_voxel_grid, prune_mask)
    
    # Step 2: Reference Graph Construction
    # For each unpruned voxel, select one adjacent voxel as reference.
    # 7 possible neighbors are considered.
    reference_graph = {}
    for voxel in unpruned_voxels:
        neighbors = get_7_adjacent_neighbors(voxel)
        # Select best reference based on correlation/prediction error
        ref_voxel = select_best_reference(neighbors)
        reference_graph[voxel.id] = ref_voxel.id
        
    # Step 3: RD Loss Finetuning
    # Estimate bit cost based on reference graph and current grid state
    # Minimize RD Loss = Distortion + lambda * Bitrate
    voxel_grid = finetune_with_rd_loss(uncompressed_voxel_grid, reference_graph)
    
    # Step 4: Coarse Residual Generation
    # Quantize and predict for non-critical voxels
    coarse_residuals = {}
    for voxel in unpruned_voxels:
        if not critical_mask[voxel.id]:
            ref_id = reference_graph[voxel.id]
            predicted_value = voxel_grid[ref_id]
            actual_value = voxel_grid[voxel.id]
            coarse_residuals[voxel.id] = quantize(actual_value - predicted_value)
            
    # Step 5: Post-Finetuning for Critical Voxels
    # Freeze non-critical voxels
    frozen_grid = voxel_grid.copy()
    for voxel in unpruned_voxels:
        if not critical_mask[voxel.id]:
            frozen_grid[voxel.id] = voxel_grid[voxel.id] # Freeze
            
    # Finetune only critical voxels to minimize residual error
    fine_residuals = {}
    for voxel in unpruned_voxels:
        if critical_mask[voxel.id]:
            ref_id = reference_graph[voxel.id]
            # Note: Reference might be critical or non-critical. 
            # If non-critical, use frozen grid value. If critical, use updated value.
            ref_value = frozen_grid[ref_id] if not critical_mask[ref_id] else voxel_grid[ref_id]
            actual_value = voxel_grid[voxel.id]
            fine_residuals[voxel.id] = quantize(actual_value - ref_value)
            
    # Step 6: Encoding
    # Arithmetic coding for residuals and indexes
    bitstream = []
    bitstream.append(encode_arithmetic(reference_graph))
    bitstream.append(encode_arithmetic(coarse_residuals))
    bitstream.append(encode_arithmetic(fine_residuals))
    
    # LZ77 for small syntax elements
    bitstream.append(encode_lz77(prune_mask))
    bitstream.append(encode_lz77(critical_mask))
    bitstream.append(encode_lz77(get_non_prune_density_grid(uncompressed_voxel_grid, prune_mask)))
    bitstream.append(encode_lz77(get_shallow_mlp_weights(voxel_grid)))
    bitstream.append(encode_lz77(metadata))
    
    return bitstream
```

**Parameters:**
*   **Pruning Ratio:** Approximately 90% of unimportant voxels are pruned [S1].

**Edge Cases and Pitfalls:**
*   **Reference Selection:** The choice of reference from seven neighbors impacts the residual magnitude and thus the bitrate. Poor selection leads to higher distortion or bitrate [S1].
*   **Critical Voxels:** The definition of "critical" is not explicitly detailed in the sources, but it is crucial for the two-stage finetuning. Incorrect selection may lead to suboptimal quality [S1].
*   **Precision:** Quantization of residuals is necessary for compression. The sources do not specify the bit depth for quantization, which is a potential implementation gap [G].

## Operating characteristics
*   **Compression Performance:** Achieves 32% bit saving compared to VQRF on multiple representative test datasets [S1].
*   **Training Time:** Comparable to the baseline DirectVoxGO pipeline [S1].
*   **Strengths:** Effective removal of spatial redundancy through predictive coding; joint optimization of rate and distortion [S1].
*   **Limitations:** Relies on the initial quality of the uncompressed grid from DirectVoxGO; the two-stage finetuning adds complexity [S1].

## Background
SPC-NeRF was proposed to address the high computational and storage complexity of NeRF models represented by Explicit Voxel Grids [S1]. It builds on the concept of spatial predictive coding, which exploits statistical dependence between spatially adjacent samples, analogous to image compression techniques [S1]. The method was developed to improve coding efficiency by jointly optimizing compression ratio and distortion [S1].

## Sources
[S1] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
