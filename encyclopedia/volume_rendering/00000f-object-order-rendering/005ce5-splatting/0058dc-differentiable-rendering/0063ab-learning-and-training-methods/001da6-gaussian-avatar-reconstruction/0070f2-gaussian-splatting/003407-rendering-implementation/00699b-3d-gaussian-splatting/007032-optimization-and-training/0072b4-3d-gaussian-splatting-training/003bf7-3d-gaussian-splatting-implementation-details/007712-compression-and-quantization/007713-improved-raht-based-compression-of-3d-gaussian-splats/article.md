# Improved RAHT-based Compression of 3D Gaussian Splats

Also called: RAHT-based Compression for 3DGS

## Overview
This method compresses pre-trained 3D Gaussian Splatting (3DGS) models by treating them as point clouds with rich attributes, applying the Region-Adaptive Hierarchical Transform (RAHT) to decorrelate data and promote sparsity [S1]. It distinguishes itself from sibling methods like [Codebook Quantization](23296-codebook-quantization.md) and [MesonGS](23296-codebook-quantization.md) by systematically applying transform coding to *all* Gaussian attributes rather than a subset, and by operating on unstructured pre-trained models without requiring changes to the training pipeline or additional side information [S1].

## How it works
**Inputs:** Pre-trained 3DGS parameters: positions $\mu \in \mathbb{R}^3$, scales $s \in \mathbb{R}^3$, rotations $q \in \mathbb{R}^4$ (quaternions), opacities $\alpha \in \mathbb{R}$, and spherical harmonic (SH) coefficients for view-dependent color.
**Outputs:** A compressed bitstream containing geometry, attributes, and metadata.

**Data Structures:**
1.  **Gaussian Set:** A list of $N$ Gaussians, each defined by $\mu, s, q, \alpha, \text{SH}$.
2.  **Octree:** A hierarchical spatial structure used to voxelize positions and define the RAHT hierarchy. Depth $d$ is selected to preserve most Gaussians while minimizing rendering distortion [S1].
3.  **Attribute Blocks:** Each attribute channel is partitioned into blocks based on Morton ordering (Z-order curve) to ensure spatially coherent grouping of Gaussians [S1].

**Steps:**

1.  **Gaussian Pruning:**
    Remove redundant Gaussians using importance-based pruning [S1]. Compute an importance score $I$ for each Gaussian as the product of a view-dependent score $I_d$ and a view-independent score $I_i$ derived from the Gaussian’s volume [S1]. Discard Gaussians with the lowest scores [S1].

2.  **Geometry Compression:**
    Voxelize the 3D positions $\mu$ of the remaining Gaussians using an Octree structure [S1]. Encode the geometry bitstream using the MPEG GPCC codec [S1].

3.  **Attribute Pre-processing:**
    For merged Gaussians (if any), average their attributes [S1]. Replace the rotation quaternion $q \in \mathbb{R}^4$ with an Euler angle representation $e \in \mathbb{R}^3$ to reduce dimensionality [S1].

4.  **Fine-Tuning:**
    Perform a short fine-tuning of the 3DGS parameters to balance the rate-distortion (RD) tradeoff and improve compression efficiency by promoting sparsity in the transform domain [S1]. [G] This step typically involves a few iterations of gradient descent on the training loss with a regularization term encouraging coefficient sparsity, using the pruned Gaussian set.

5.  **RAHT Transform:**
    Partition each attribute channel into blocks based on Morton ordering [S1]. Apply the RAHT transform independently to each attribute channel [S1]. RAHT operates along the Octree hierarchy, applying successive transforms across spatial axes to decorrelate neighboring attribute values [S1].

6.  **Channel-wise Bit Allocation:**
    Determine the quantization parameter for each attribute channel based on its rendering impact [S1]. Analyze the RD curve for each attribute $i$:
    $$ D_{\text{render}} = f(R_i), \quad i \in [1, n_{\text{attr}}] $$
    where $D_{\text{render}}$ is rendering distortion and $R_i$ is the per-channel bitrate [S1].
    To determine $R_i$, test quantization bits $b \in \{2, 4, 6, 8, 16, 32\}$ for attribute $i$ while keeping all other attributes $j \neq i$ fixed at 8 bits [S1]. Select the bitrate that achieves acceptable quality with minimal bits, noting that attributes saturate at different bitrates (e.g., rotation saturates around 8 bpp, while SH coefficients may saturate earlier) [S1].

7.  **Quantization:**
    Quantize each block independently using uniform scalar quantizers with the distinct quantization parameter chosen per channel [S1].

8.  **Entropy Coding:**
    Entropy code all data (geometry, attributes, metadata) using the deflate algorithm to produce the final bitstream [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_3dgs(gaussians, scene_data):
    """
    Compress 3D Gaussian Splats using RAHT-based compression.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   mu (3,), s (3,), q (4,), alpha (1,), sh (coeffs)
        scene_data: Training data for pruning and fine-tuning.
        
    Returns:
        bitstream: Compressed data.
    """
    # Step 1: Pruning
    # Compute importance score I = I_d * I_i
    # I_d: view-dependent score (rendering relevance)
    # I_i: view-independent score (volume)
    importance_scores = compute_importance(gaussians, scene_data)
    pruned_gaussians = prune_by_score(gaussians, importance_scores, threshold=0.01) # [G] threshold example
    
    # Step 2: Geometry Compression
    octree = build_octree(pruned_gaussians.mu)
    geometry_bitstream = encode_mpeg_gpcc(octree)
    
    # Step 3: Attribute Pre-processing
    # Merge Gaussians if necessary (not detailed in source, assumed minimal)
    # Replace quaternion with Euler angles
    for g in pruned_gaussians:
        g.e = quaternion_to_euler(g.q)
        g.q = None # Remove quaternion to save space
        
    # Step 4: Fine-Tuning
    # Short fine-tuning to promote sparsity
    pruned_gaussians = fine_tune_gaussians(pruned_gaussians, scene_data, iterations=10) # [G] iterations example
    
    # Step 5: RAHT Transform
    # Partition attributes into blocks based on Morton ordering
    attribute_blocks = {}
    for attr_name in ['s', 'alpha', 'e', 'sh']:
        values = [getattr(g, attr_name) for g in pruned_gaussians]
        blocks = partition_by_morton(values, octree)
        attribute_blocks[attr_name] = apply_raht(blocks, octree)
        
    # Step 6: Channel-wise Bit Allocation
    # Determine optimal bits per channel based on RD curves
    # D_render = f(R_i)
    # Test b in {2, 4, 6, 8, 16, 32} for each attribute, others fixed at 8 bits
    bit_allocation = {}
    for attr_name in attribute_blocks.keys():
        rd_curve = compute_rd_curve(attribute_blocks[attr_name], scene_data)
        bit_allocation[attr_name] = select_optimal_bits(rd_curve, target_quality=0.95) # [G] target_quality example
        
    # Step 7: Quantization
    quantized_blocks = {}
    for attr_name, blocks in attribute_blocks.items():
        b = bit_allocation[attr_name]
        quantized_blocks[attr_name] = uniform_scalar_quantize(blocks, bits=b)
        
    # Step 8: Entropy Coding
    metadata = {
        'num_gaussians': len(pruned_gaussians),
        'octree_depth': octree.depth,
        'bit_allocation': bit_allocation
    }
    bitstream = encode_deflate(geometry_bitstream, quantized_blocks, metadata)
    
    return bitstream
```

**Edge Cases and Pitfalls:**
*   **Precision:** Position accuracy is critical; positions are encoded via Octree (MPEG GPCC) rather than RAHT to preserve geometric fidelity [S1].
*   **Saturation:** Attributes saturate at different bitrates; applying uniform quantization across all attributes leads to suboptimal compression [S1].
*   **Quaternion vs. Euler:** Replacing quaternions with Euler angles reduces dimensionality but may introduce gimbal lock issues if not handled carefully during fine-tuning [S1]. [G] Ensure Euler angle representation is consistent (e.g., Z-Y-X order) and handle discontinuities during interpolation or averaging.

## Subtopics

### Codebook Quantization
Uses a learned codebook to quantize attributes, often requiring vector quantization. Choose when attribute distributions are highly non-Gaussian and can be well-approximated by a small codebook. See [Codebook Quantization](23296-codebook-quantization.md).

### MesonGS
Applies RAHT to a subset of attributes and vector quantizes the rest. Choose when some attributes (e.g., SH) are better suited for vector quantization. See [MesonGS](23296-codebook-quantization.md).

## Operating characteristics
*   **Time Complexity:** Dominated by Octree construction, RAHT transform, and fine-tuning. Fine-tuning adds training time, but compression itself is efficient due to standard codecs (MPEG GPCC, deflate) [S1].
*   **Memory Complexity:** Depends on the Octree depth and the number of Gaussians. Pruning reduces the number of Gaussians, lowering memory usage [S1].
*   **Performance:** Achieves competitive compression ratios while maintaining rendering quality. The method is compatible with existing standard coding tools [S1].
*   **Strengths:** No training pipeline changes required; systematic bit allocation; handles all attributes with RAHT [S1].
*   **Limitations:** Fine-tuning adds computational overhead; Euler angle representation may be less robust than quaternions for certain rotations [S1].
*   **When to use:** When compressing pre-trained 3DGS models without retraining is required, and high compatibility with standard codecs is needed [S1].
*   **When not to use:** When training pipeline modifications are acceptable and more aggressive compression via structured representations (e.g., Scaffold-GS) is desired [S1].

## Background
The method was proposed in "Improved RAHT-based Compression of 3D Gaussian Splats" (2025) [S1]. It builds on the observation that 3DGS can be interpreted as a point cloud with rich attributes, leveraging the well-established RAHT transform from point cloud compression standards [S1]. Previous attempts using RAHT on 3DGS showed limited efficiency due to redundancy and unstructured data; this method addresses this with fine-tuning and systematic bit allocation [S1]. It contrasts with Scaffold-GS, which requires significant training pipeline changes, and MesonGS, which uses a hybrid approach [S1].

## Sources
[S1] Improved RAHT-based Compression of 3D Gaussian Splats, https://doi.org/10.1109/pcs65673.2025.11417628, fetched 2026-10-26
