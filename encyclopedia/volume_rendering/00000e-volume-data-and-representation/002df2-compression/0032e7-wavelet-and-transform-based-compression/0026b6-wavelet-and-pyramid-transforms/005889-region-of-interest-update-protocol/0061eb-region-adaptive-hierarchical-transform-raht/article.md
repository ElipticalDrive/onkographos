# Region Adaptive Hierarchical Transform (RAHT)

Also called: MPEG PCC TMC1 Attribute Coding, SLF Coefficient Compression via RAHT

## Overview
Region Adaptive Hierarchical Transform (RAHT) is a bottom-up wavelet transform that reorganizes color or scalar attributes along an octree structure by pairing neighboring voxels to produce low-frequency (DC) and high-frequency coefficients [S1]. It sets itself apart from graph-based transforms by relying on simple pairwise operations rather than costly eigen-decompositions, making it computationally efficient and suitable for real-time compression of attributes with strong local correlation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of scalar values (e.g., color components, SLF coefficients) associated with voxels in an octree. Each voxel has an associated weight representing the number of underlying fine-grained voxels [S1].
*   **Output:** A stream of quantized, entropy-coded coefficients. The decoder reconstructs the original attributes via inverse operations [S2].

**Data Structures**
The method operates on an octree hierarchy. Each node in the octree represents a voxel.
*   **Voxel Weight ($w$):** An integer value indicating the number of underlying fine-grained voxels represented by the current voxel [S1].
*   **Attribute Value ($F$):** The scalar value associated with the voxel (e.g., a color channel component) [S2].

**Steps**
1.  **Pairing:** At each level of the octree, spatially neighboring voxels are paired. The pairing is determined by the weights of the voxels, which reflect the local density [S1].
2.  **Transform:** A 2-point Haar transform is applied to each pair of neighboring voxels. This produces one DC coefficient (low-frequency) and one high-frequency coefficient [S1, S2].
3.  **Propagation:** The DC coefficients are propagated upward to the parent level for further merging with other DC coefficients [S1].
4.  **Encoding:** The high-frequency coefficients are encoded immediately at the current level [S1].
5.  **Quantization:** The resulting coefficients are quantized using a step size $Q$ [S2].
6.  **Entropy Coding:** The quantized coefficients are entropy encoded. Sub-band parameters are encoded using a Run-Length Golomb-Rice (RLGR) coder, while the coefficients themselves use an arithmetic coder with a Laplacian model [S1].

**Formulas**
The 2-point Haar transform for a pair of values $F_1$ and $F_2$ with weights $w_1$ and $w_2$ produces a DC coefficient $F_{DC}$ and a high-frequency coefficient $F_{HF}$. While the exact transform matrix weights are not explicitly detailed in the provided notes, the notes state that the transform adapts to local density by adjusting the transform matrix to the weights [S1].

Quantization is defined as [S2]:
$$
\hat{F}_i = \text{Round}\left(\frac{F_i}{Q}\right)
$$
Where:
*   $F_i$ is the RAHT transformed coefficient.
*   $Q$ is the quantization step-size.
*   $\hat{F}_i$ is the quantized coefficient.

**Parameters**
*   **$Q$ (Quantization step-size):** Controls the trade-off between bitrate and reconstruction quality. Larger values of $Q$ yield lower bitrate but inferior reconstruction quality [S2]. Typical values are not given in the sources.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raht_encode(attributes, octree, Q):
    """
    Compresses attributes using RAHT.
    attributes: dict mapping voxel_id -> (value, weight)
    octree: structure defining parent-child relationships and spatial neighbors
    Q: quantization step-size
    """
    quantized_coeffs = []
    
    # Process octree from bottom-up (leaves to root)
    for level in reversed(octree.levels):
        # Get voxels at this level
        voxels = get_voxels_at_level(octree, level)
        
        # Pair neighboring voxels
        # Note: Pairing logic depends on spatial adjacency and weights
        pairs = pair_neighboring_voxels(voxels)
        
        for v1, v2 in pairs:
            val1, w1 = attributes[v1.id]
            val2, w2 = attributes[v2.id]
            
            # Apply 2-point Haar transform adapted to weights
            # The exact weighting formula for the Haar transform is not specified in sources
            # [G] Standard weighted Haar transform for values x, y with weights w_x, w_y:
            # DC = (w_x * x + w_y * y) / (w_x + w_y)
            # HF = (x - y) / sqrt(w_x + w_y) or similar normalization
            # However, sources only mention "adjusting the transform matrix to the weights"
            # We implement a generic weighted average for DC and difference for HF
            # as a placeholder for the specific matrix adjustment mentioned in [S1]
            
            total_weight = w1 + w2
            if total_weight == 0:
                continue
                
            # DC coefficient (low-frequency)
            dc = (w1 * val1 + w2 * val2) / total_weight
            
            # High-frequency coefficient
            # [G] Standard difference scaled by weight. Exact normalization not in sources.
            hf = (val1 - val2) / sqrt(total_weight)
            
            # Quantize
            q_dc = round(dc / Q)
            q_hf = round(hf / Q)
            
            # Encode high-frequency immediately
            quantized_coeffs.append(('HF', v1.id, v2.id, q_hf))
            
            # Propagate DC upward
            # Update parent voxel's attribute with the DC value
            # The weight of the parent is the sum of children weights (already w1+w2)
            # If parent already exists, merge with existing DC? 
            # Sources say "DC coefficients propagate upward for further merging"
            # [G] Assuming parent accumulates the DC value weighted by its own weight structure
            # For simplicity, we store the DC value in the parent's attribute for the next level
            if v1.parent_id in attributes:
                # Merge with existing parent attribute if it exists (from other children)
                # This requires a more complex data structure than simple dict
                # [G] Placeholder for merging logic
                pass
            else:
                attributes[v1.parent_id] = (dc, total_weight)

    # Encode the root DC coefficients
    # ...
    return quantized_coeffs

def raht_decode(quantized_coeffs, octree, Q):
    """
    Reconstructs attributes from RAHT encoded data.
    """
    # Inverse operations: entropy decoding, inverse quantization, inverse RAHT
    # [G] Implementation details for entropy decoding and inverse quantization
    # are standard and not specific to RAHT's core transform logic.
    pass
```

**Edge Cases and Pitfalls**
*   **Precision:** Quantization introduces loss. The choice of $Q$ directly impacts quality [S2].
*   **Boundaries:** The method relies on spatial neighbors. Voxels at the boundary of the octree or point cloud may have fewer neighbors, affecting pairing [S1].
*   **Degenerate Input:** Voxels with zero weight are skipped [S1].

## Subtopics

### MPEG PCC TMC1 Attribute Coding
The core of the MPEG Point Cloud Codec Test Model Category 1 (TMC1), using RAHT for attribute compression [S2].

### SLF Coefficient Compression via RAHT
Application of RAHT to compress Surface Light Field (SLF) coefficients by treating them as point cloud attributes [S2].

## Operating characteristics
*   **Time Complexity:** The transform relies on simple pairwise operations, avoiding costly eigen-decompositions required by graph-based transforms, making it computationally efficient and suitable for real-time compression [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Achieves compression performance comparable to state-of-the-art methods for point-cloud compression, but at significantly lower complexity [S1].
*   **Strengths:** Adapts to local density by adjusting the transform matrix to weights, effectively preserving small-scale details while reducing redundancy [S1]. Suitable for attributes with strong local correlation, such as color or density [S1].
*   **Limitations:** Relies on octree structure; performance may degrade if local correlation is weak [S1].
*   **When to use:** When compressing point cloud attributes (color, density) with strong local correlation and requiring real-time performance [S1, S2].
*   **When not to use:** When graph-based transforms are preferred for specific geometric structures, or when higher compression ratios are needed at the cost of complexity [S1].

## Background
RAHT was developed as part of the MPEG Point Cloud Compression (PCC) standard, specifically as the core of the Test Model Category 1 (TMC1) [S2]. It is a bottom-up transform that reorganizes color attributes along an octree [S1]. The method is particularly effective for attributes that exhibit strong local correlation, such as color or density, and has been adapted for use in 3D Gaussian Splatting compression [S1].

## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S2] Surface Light Field Compression Using a Point Cloud Codec, https://doi.org/10.1109/jetcas.2018.2883479, fetched 2026-10-28
