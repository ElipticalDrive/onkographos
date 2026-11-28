# VQ with Undefined Data

Also called: Partial VQ, VQ for undefined residuals

## Overview
This method extends vector quantization (VQ) to handle residual blocks where some data values are undefined, such as those outside the defined domain bounds [S1, unofficial]. It reduces quantization error for valid data by ignoring undefined components during distance calculations and centroid updates [S1, unofficial]. The approach preserves the convergence properties of standard k-means clustering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Residual vectors $V = \{v_1, \dots, v_N\}$ where each $v_i \in \mathbb{R}^D$ may contain undefined components [S1, unofficial].
- Input: Cluster centroids $C = \{c_1, \dots, c_K\}$ [G].
- Input: Validity masks $M_i \in \{0, 1\}^D$ for each vector, where $M_{i,d} = 1$ if component $d$ is defined, and $0$ otherwise [G].
- Output: Cluster assignments $A = \{a_1, \dots, a_N\}$ [G].
- Output: Updated centroids $C' = \{c'_1, \dots, c'_K\}$ [G].

**Data Structures:**
- Vectors, masks, and centroids are stored as dense arrays of size $D$ [G].

**Algorithm Steps:**
The method modifies the standard k-means clustering algorithm [S1, unofficial].

1. **Initialization:** Initialize centroids $C$ using a standard method (e.g., k-means++) [G].
2. **Assignment Step:** For each vector $v_i$, compute the squared distance to each centroid $c_j$, ignoring undefined components [S1, unofficial]:
   $$ d^2(v_i, c_j) = \sum_{d=1}^{D} M_{i,d} \cdot (v_{i,d} - c_{j,d})^2 $$
   Assign $v_i$ to the cluster $a_i$ with the minimum distance: $a_i = \arg\min_{j} d^2(v_i, c_j)$ [G].
3. **Update Step:** For each cluster $j \in \{1, \dots, K\}$, compute the new centroid mean ignoring undefined components of the assigned vectors [S1, unofficial]:
   $$ c'_{j,d} = \frac{\sum_{i \in S_j} M_{i,d} \cdot v_{i,d}}{\sum_{i \in S_j} M_{i,d}} $$
   where $S_j = \{i \mid a_i = j\}$ is the set of indices assigned to cluster $j$ [G]. If the denominator is zero, $c'_{j,d}$ remains unchanged [G].
4. **Convergence:** Repeat steps 2 and 3 until assignments $A$ do not change or centroid changes fall below a threshold [G].

**Parameters:**
- $K$: Number of clusters. Not given in the sources.
- $\tau$: Convergence threshold. Not given in the sources.

**Edge Cases and Pitfalls:**
- If a vector has all components undefined, distance is zero to all centroids; assignment is arbitrary [G].
- If a cluster has no assigned vectors, the update step is skipped [G].
- Precision issues may arise if the number of defined components is very small [G].

## Subtopics

### [K-means Parameter Decompression](11821-k-means-parameter-decompression.md)
Reconstructs a parameter tensor from compressed K-means centroids and cluster labels by dequantizing and remapping values. It reverses the quantization and indexing steps of the compression phase. Choose this when decompressing parameter data that was quantized using this method.

## Operating characteristics
- **Strengths:** Reduces quantization error for valid data by ignoring undefined components. Preserves the convergence properties of k-means clustering [S1, unofficial].
- **Limitations:** Requires validity masks for each vector. Performance may degrade if a large fraction of components are undefined [G].
- **When to use:** When processing residual blocks with undefined values, such as those extending beyond domain bounds [S1, unofficial].
- **When not to use:** When all data components are defined, as standard VQ is simpler [G].
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.

## Background
This method extends k-means clustering to partially defined data, leveraging the property that k-means preserves convergence when ignoring undefined components during distance and mean calculations [S1, unofficial]. It is used within brood-based vector quantization to compress residual blocks that extend beyond defined domain bounds [S1, unofficial].

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
