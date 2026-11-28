# Region Adaptive Hierarchical Transform (RAHT)

Also called: MPEG PCC TMC1 Attribute Coding, SLF Coefficient Compression via RAHT

## Overview
Region Adaptive Hierarchical Transform (RAHT) is a bottom-up transform that reorganizes color or scalar attributes along an octree by pairing neighboring voxels to produce low-frequency (DC) and high-frequency coefficients [S1]. It distinguishes itself from graph-based transforms by relying on simple pairwise operations rather than costly eigen-decompositions, making it computationally efficient and suitable for real-time compression of attributes with strong local correlation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of scalar values (e.g., color components or Surface Light Field coefficients) associated with octree nodes, where each node carries a weight representing the number of underlying fine-grained voxels [S1, S2].
*   **Output:** A stream of quantized, entropy-coded coefficients [S1]. The decoder reconstructs the original attributes via entropy decoding, inverse quantization, and inverse RAHT [S2].

**Data Structures**
The method operates on an octree hierarchy where each node represents a voxel [G].
*   **Voxel Weight ($w$):** An integer value indicating the number of underlying fine-grained voxels represented by the current voxel [S1].
*   **Attribute Value ($F$):** The scalar value associated with the octree node [S2].

**Steps**
1.  **Pairing:** At each level of the octree, spatially neighboring voxels are paired according to their weights [S1].
2.  **Transform:** A 2-point Haar transform is applied to each pair of neighboring voxels, producing one DC coefficient and one high-frequency coefficient [S1, S2].
3.  **Propagation:** The DC coefficients are propagated upward to the parent level for further merging with other DC coefficients [S1].
4.  **Encoding:** The high-frequency coefficients are encoded immediately at the current level [S1].
5.  **Quantization:** The resulting coefficients are quantized using a step size $Q$ [S2].
6.  **Entropy Coding:** The quantized coefficients are entropy encoded using an arithmetic coder with a Laplacian model, while sub-band parameters are encoded using a Run-Length Golomb-Rice (RLGR) coder [S1].

**Formulas**
The 2-point Haar transform for a pair of values $F_1$ and $F_2$ with weights $w_1$ and $w_2$ produces a DC coefficient $F_{DC}$ and a high-frequency coefficient $F_{HF}$ [S1]. The transform matrix is adjusted to the weights to adapt to local density [S1]. A standard weighted implementation is [G]:
$$
F_{DC} = \frac{w_1 F_1 + w_2 F_2}{w_1 + w_2}, \quad F_{HF} = \frac{F_1 - F_2}{\sqrt{w_1 + w_2}}
$$
Quantization is defined as [S2]:
$$
\hat{F}_i = \text{Round}\left(\frac{F_i}{Q}\right)
$$
Where:
*   $F_i$ is the RAHT transformed coefficient [S2].
*   $Q$ is the quantization step-size [S2].
*   $\hat{F}_i$ is the quantized coefficient [S2].

**Parameters**
*   **$Q$ (Quantization step-size):** Controls the trade-off between bitrate and reconstruction quality [S2]. Larger values of $Q$ yield lower bitrate but inferior reconstruction quality [S2]. Typical values are not given in the sources.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def raht_encode(attributes, octree_levels, octree_neighbors, octree_parent, Q):
    """
    Compresses attributes using RAHT.
    attributes: dict mapping node_id -> (value, weight)
    octree_levels: list of lists, node_ids per level from bottom to top
    octree_neighbors: dict mapping node_id -> list of neighbor_ids
    octree_parent: dict mapping node_id -> parent_node_id
    Q: quantization step-size
    """
    quantized_coeffs = []
    current_attrs = dict(attributes)

    for level in octree_levels:
        processed = set()
        for node_id in level:
            if node_id in processed:
                continue
            neighbors = octree_neighbors.get(node_id, [])
            pair_id = None
            for n_id in neighbors:
                if n_id not in processed and n_id in current_attrs:
                    pair_id = n_id
                    break
            if pair_id is None:
                processed.add(node_id)
                continue

            processed.add(node_id)
            processed.add(pair_id)

            val1, w1 = current_attrs[node_id]
            val2, w2 = current_attrs[pair_id]
            total_weight = w1 + w2

            if total_weight == 0:
                continue

            dc = (w1 * val1 + w2 * val2) / total_weight
            hf = (val1 - val2) / math.sqrt(total_weight)

            q_dc = round(dc / Q)
            q_hf = round(hf / Q)

            quantized_coeffs.append(('HF', node_id, pair_id, q_hf))

            parent_id = octree_parent.get(node_id)
            if parent_id is not None:
                current_attrs[parent_id] = (dc, total_weight)

    return quantized_coeffs

def raht_decode(quantized_coeffs, octree_levels, octree_neighbors, octree_parent, Q):
    """
    Reconstructs attributes from RAHT encoded data.
    """
    # Inverse operations: entropy decoding, inverse quantization, inverse RAHT [S2]
    # [G] Standard arithmetic/Laplacian decoding and RLGR parsing are assumed.
    # Reconstruct values level-by-level from root to leaves using stored HF coefficients.
    pass
```

**Edge Cases and Pitfalls**
*   **Precision:** Quantization introduces loss, and the choice of $Q$ directly impacts quality [S2].
*   **Degenerate Input:** Voxels with zero total weight are skipped to avoid division by zero [G].

## Subtopics

### MPEG PCC TMC1 Attribute Coding
The core of the MPEG Point Cloud Codec Test Model Category 1 (TMC1), using RAHT for attribute compression [S2].

### SLF Coefficient Compression via RAHT
Application of RAHT to compress Surface Light Field (SLF) coefficients by treating them as point cloud attributes [S2].

### [Rate-Distortion Optimized RAHT](21040-rate-distortion-optimized-raht.md)
Rate-Distortion Optimized RAHT (RDO RAHT) is a standard-compatible quantization framework for the Region Adaptive Hierarchical Transform (RAHT) in G-PCC, designed to optimize bit allocation for high-dynamic-range attributes such as those in 3D Gaussian Splatting (3DGS). It distinguishes itself from standard RAHT by introducing a configurable fixed-point precision extension to mitigate numerical error floors and deriving an analytical rate-distortion model that allocates bits according to the hierarchical contribution of RAHT layers, achieving significant BD-rate gains without requiring neural training.

## Operating characteristics
*   **Time Complexity:** The transform relies on simple pairwise operations, avoiding costly eigen-decompositions required by graph-based transforms, making it computationally efficient and suitable for real-time compression [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Achieves compression performance comparable to state-of-the-art methods for point-cloud compression, but at significantly lower complexity [S1].
*   **Strengths:** Adapts to local density by adjusting the transform matrix to weights, effectively preserving small-scale details while reducing redundancy [S1]. Suitable for attributes with strong local correlation, such as color or density [S1].
*   **Limitations:** Relies on octree structure [S1].
*   **When to use:** When compressing point cloud attributes (color, density) with strong local correlation and requiring real-time performance [S1, S2].
*   **When not to use:** Not given in the sources.

## Background
RAHT was developed as part of the MPEG Point Cloud Compression (PCC) standard, specifically as the core of the Test Model Category 1 (TMC1) [S2]. It is a bottom-up transform that reorganizes color attributes along an octree [S1]. The method is particularly effective for attributes that exhibit strong local correlation, such as color or density, and has been adapted for use in 3D Gaussian Splatting compression [S1].

## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S2] Surface Light Field Compression Using a Point Cloud Codec, https://doi.org/10.1109/jetcas.2018.2883479, fetched 2026-10-28
