# Haar Wavelet Transform on Omnitrees

Also called: nodal to wavelet transform, tensor-product Haar basis

## Overview
This method transforms the nodal representation of an omnitree into a hierarchical wavelet coefficient basis to enable efficient compression. It differs from sibling methods like [Downsplit](3404-downsplit.md) and [Coarsening Compression](3537-coarsening-compression.md) by explicitly mapping the piecewise constant leaf-node basis functions to a tensor-product Haar wavelet basis, allowing for lossless or lossy compression via coefficient pruning and tree restructuring.

## How it works
The transform maps the nodal representation of a volume, defined as a sum over leaf nodes, into a hierarchical wavelet basis.

**Inputs and Outputs:**
*   **Input:** An omnitree structure where the volume function $f$ is represented by nodal coefficients $s^v$ associated with leaf nodes $v$ (where splitting label $b(v)=0$).
*   **Output:** A set of wavelet coefficients corresponding to the same spatial domain, organized hierarchically.

**Data Structures:**
The omnitree nodes store splitting labels $\vec{b}(v)$, a vector where each component $b_j \in \{0, 1\}$ indicates whether the node is split in dimension $j$. Leaf nodes have $b(v)=\vec{0}$.

**Method Steps:**
1.  **Nodal Representation:** The volume function $f \in L^2([0,1]^d)$ is initially represented as a sum over leaf nodes $v$:
    $$ f = \sum_{v \in \text{Leaves}} s^v \phi^v $$
    where $\phi^v$ is the basis function for node $v$, defined as 1 if $x \in \text{rect}(v)$ and 0 otherwise [S1, unofficial].

2.  **Basis Transformation:** The nodal basis functions $\phi^v$ are expressed in terms of the tensor-product Haar wavelet basis. The Haar wavelet transform is applied hierarchically. For a node $v$ that is split, the constant value over the parent rect is decomposed into a coarse approximation (average) and detail coefficients (differences) for the children.

3.  **Coefficient Calculation:** For each split dimension $j$, the transform computes the average and difference of the coefficients of the two children along that dimension. In 3D, this is applied across dimensions. The specific application involves applying the 3D Haar wavelet transform to blocks of data [S2, unofficial].

4.  **Compression (Optional):** The resulting wavelet coefficients are quantized and encoded. This may involve partitioning the volume into small blocks, subdividing them into subblocks, and applying the 3D Haar wavelet transform two or three times [S2, unofficial].

**Parameters:**
*   **Number of Transform Levels:** Typically 2 or 3 for 3D data [S2, unofficial].
*   **Block Size:** The volume is partitioned into small blocks before transformation [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The transform assumes a specific domain $[0,1]^d$. Handling boundaries requires care to avoid artifacts [S1, unofficial].
*   **Precision:** Quantization introduces loss; the choice of quantization step affects compression ratio and visual quality [S2, unofficial].

## Subtopics

### Haar Wavelet Compression
A specific application where the volume is partitioned into small blocks, which are further subdivided into subblocks. A three-dimensional Haar wavelet is applied two or three times to these subblocks. The resulting coefficients are then quantized and encoded [S2, unofficial].

## Operating characteristics
*   **Complexity:** The transform is applied hierarchically. The complexity depends on the number of nodes and the depth of the omnitree. Block-based approaches have complexity proportional to the number of voxels [S2, unofficial].
*   **Performance:** Wavelet-based compression can achieve high compression ratios, especially for smooth volumes, by pruning small coefficients [S2, unofficial].
*   **Strengths:** Efficient representation of piecewise constant functions; enables both lossless and lossy compression [S1, S2, unofficial].
*   **Limitations:** Block-based approaches may introduce blocking artifacts at block boundaries [S2, unofficial].

## Background
The Haar wavelet transform on omnitrees is part of the broader field of wavelet-based volume rendering and compression. It builds on the concept of representing volumes using hierarchical data structures like omnitrees [S1, unofficial]. The specific use of Haar wavelets for block-based volume compression was explored by Insung et al. and Kim et al. [S2, unofficial].

## Sources
[S1] Towards Fully Dynamic Omnitrees: Moment-Conserving Anisotropic Compression With Wavelets, https://doi.org/10.48550/arxiv.2607.04881, fetched 2026-10-26 (unofficial)
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27 (unofficial)
