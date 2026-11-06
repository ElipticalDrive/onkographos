# Haar Wavelet Transform on Omnitrees
Also called: nodal to wavelet transform, tensor-product Haar basis
## Overview
This method transforms nodal omnitree data into wavelet coefficients to enable lossless or lossy compression via pruning and tree restructuring (downsplit). It sets itself apart from sibling methods like [Downsplit](3404-downsplit.md) and [Coarsening Compression](3537-coarsening-compression.md) by explicitly defining the mathematical basis functions and the hierarchical decomposition of piecewise constant functions over the omnitree structure [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   Input: A nodal omnitree representation of a function $f \in L^2([0,1]^d)$.
*   Output: A set of wavelet coefficients corresponding to the hierarchical decomposition.

**Data Structures:**
The omnitree is defined by splitting labels $\vec{b}(v)$ for each node $v$.
*   $\vec{b}(v)$: A vector of length $d$ (dimensionality of the domain).
*   $b_j(v) \in \{0, 1\}$: The $j$-th component indicates if the node is split in dimension $j$.
*   Leaf nodes $v$ with $\vec{b}(v) = \vec{0}$ store the base values.

**Mathematical Definition:**
The nodal representation stores $f$ as a sum over leaf nodes $v$ with $\vec{b}(v) = \vec{0}$:
$$ f = \sum_{v: \vec{b}(v)=\vec{0}} s^v \phi^v $$
where $\phi^v$ is the basis function defined as:
$$ \phi^v(x) = \begin{cases} 1 & \text{if } x \in \text{rect}(v) \\ 0 & \text{otherwise} \end{cases} $$
Here, $\text{rect}(v)$ is the hyper-rectangular region associated with leaf node $v$. The coefficients $s^v$ are the nodal values.

**Transformation Steps:**
1.  **Identify Leaves:** Locate all nodes $v$ in the omnitree where $\vec{b}(v) = \vec{0}$.
2.  **Construct Basis:** For each leaf $v$, define the indicator function $\phi^v$ over its rectangular domain $\text{rect}(v)$.
3.  **Compute Nodal Values:** Determine the scalar value $s^v$ associated with each leaf. In the context of compression, these are the raw data values stored at the leaves.
4.  **Wavelet Decomposition:** Apply the Haar wavelet transform to the nodal coefficients. This involves computing differences between parent and child nodes or between adjacent blocks, depending on the specific hierarchical level. The transform converts the nodal representation (sum of $\phi^v$) into a wavelet representation (sum of wavelet basis functions).
    *   *Note:* The exact algorithm for converting nodal coefficients to wavelet coefficients on an omnitree is not fully detailed in the provided sources, but it relies on the hierarchical structure defined by $\vec{b}(v)$. The transform exploits the piecewise constant nature of the basis functions [S1, unofficial].
5.  **Coefficient Storage:** Store the resulting wavelet coefficients. Small coefficients can be pruned for lossy compression.

**Parameters:**
*   $d$: Dimensionality of the volume data.
*   $\vec{b}(v)$: Splitting labels, determining the resolution and anisotropy of the omnitree.

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors may accumulate during the transform and inverse transform.
*   **Boundary Conditions:** The definition assumes the domain $[0,1]^d$. Handling boundaries requires care to ensure $\phi^v$ is correctly defined.
*   **Degenerate Input:** Nodes with $\vec{b}(v) \neq \vec{0}$ are not leaves and do not directly contribute to the nodal sum; they define the structure for refinement.

## Subtopics

### [Haar Wavelet Compression](5378-haar-wavelet-compression.md)
A 3D compression scheme for interactive visualization of very large volume data, designed to allow random access to parts of the volume without decompressing the entire dataset. It sets itself apart from earlier wavelet compression schemes by combining a hierarchical Haar wavelet transform with a specific block-subdivision storage layout that enables efficient random access.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables lossless or lossy compression via pruning and tree restructuring. Supports anisotropic compression through the omnitree structure.
*   **Limitations:** The specific performance metrics and scaling behavior are not provided in the sources.
*   **When to use:** When compressing large volume data with anisotropic features using an omnitree representation.
*   **When not to use:** When random access without decomposition is not required, or when the data does not fit the piecewise constant model well.
## Background
The Haar wavelet transform on omnitrees is part of the broader research on fully dynamic omnitrees and moment-conserving anisotropic compression with wavelets [S1, unofficial]. The method builds on the concept of representing piecewise constant functions hierarchically using omnitrees.
## Sources
[S1] Towards Fully Dynamic Omnitrees: Moment-Conserving Anisotropic Compression With Wavelets, https://doi.org/10.48550/arxiv.2607.04881, fetched 2026-10-26 (unofficial)
