# Sparse Convolution
Also called: Sparse Conv
## Overview
Sparse convolution is a variant of standard convolution where only a subset of input elements, defined by a support set of indices, is used to improve computational efficiency [S1]. It differs from standard convolution by selecting specific input elements rather than processing all elements in the kernel window [S1]. This method serves as a core operator for sparse VDB grids, supporting various sparsity levels and feature depths [S2].

## How it works
**Inputs and Outputs:**
- Input tensor $x$ (sparse data).
- Convolutional filter $w$.
- Support set $c$ (indices specifying which elements of $x$ are used).
- Output tensor $y$.

**Data Structures:**
The method relies on the support set $c$ to index into the input tensor. In implementations derived from tree-based structures (like NanoVDB), the support indices are derived from the tree structure (e.g., leaf nodes or bricks) rather than a spatial hash [S2].

**Formula:**
The output $y$ is computed as:
$$ y = x[c] * w $$
where $*$ denotes the convolution operation, and $x[c]$ is the subset of elements from $x$ specified by the support $c$ [S1].

**Parameters:**
- **Input/Output Depth:**
  - Low-depth: 8 in, 16 out [S2].
  - Medium-depth: 32 in, 32 out [S2].
  - High-depth: 128 in, 128 out [S2].
- **Sparsity Regimes:**
  - Highly sparse: <20% leaf node occupancy [S2].
  - Moderate: 20-40% occupancy [S2].
  - Higher density: >40% occupancy [S2].

**Steps:**
1. Define the support set $c$, which specifies the indices of the input elements to be used [S1].
2. Select the relevant input elements from $x$ using the indices in $c$ [S1].
3. Apply the convolutional filter $w$ to the selected elements $x[c]$ [S1].
4. Sum the results to produce the output $y$ [S1].

**Implementation Variants (from [S2]):**
- **IGEMM:** An adaptation of SpConv v2 using a tree-derived indexing scheme instead of a spatial hash. This is the default for training to avoid specializing gradient computation for filter coefficients [S2].
- **Leaf:** Local densification at the leaf-node level (Scenario 1).
- **Brick:** Local densification at a $4 \times 2 \times 2$ "brick" level (Scenario 2) [S2].
- **LGGS:** Shared-memory Local Gather-GEMM-Scatter paradigm (Scenario 3), used only for high-depth convolution operations [S2].

**Pitfalls:**
- For training, the IGEMM option is used by default to avoid further specialization of the gradient computation for filter coefficients [S2].
- Standard sparse convolution may suffer from irregular memory access; optimized variants address this by creating locally densified windowed views [S2].

## Subtopics

### [Blocked Gather-GEMM-Scatter Sparse Convolution](9865-blocked-gather-gemm-scatter-sparse-convolution.md)
An optimization for sparse convolution that minimizes wasted MAC operations and global memory traffic by blocking the gather-GEMM-scatter operation on contiguous subsequences of output indices from a VDB data structure, using shared memory for intermediate accumulation [9865-blocked-gather-gemm-scatter-sparse-convolution.md].

### TensorCore Convolution
Convolution using tensorcores, leveraging efficient construction of locally densified, windowed views into the sparse data to enable data regularity and aggressive utilization of tensorcores for exceptional compute efficiency [S2]. Unlike standard sparse convolution which may suffer from irregular memory access, this method creates locally densified windowed views to exploit data regularity for tensorcore acceleration [S2].

## Operating characteristics
Not given in the sources.

## Background
Sparse convolutions are used in deep learning to allow for more efficient use of computation resources and improve the performance of convolutional neural networks [S1]. The specific formulation $y = x[c] * w$ is detailed in the context of SPARF (Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images) [S1], with implementation references to MinkowskiNetwork [S1]. The integration of these operators into VDB structures is part of the fVDB framework, which provides a comprehensive API for spatial intelligence and GPU-optimized fast algorithms [S2].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
[S2] fVDB: A Deep-Learning Framework for Sparse, Large-Scale, and High-Performance Spatial Intelligence, https://research.nvidia.com/labs/prl/publication/williams2024fvdb/, fetched 2026-10-30
