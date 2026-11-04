# CP Decomposition
Also called: CANDECOMP/PARAFAC, CANDECOMP/PARAFAC decomposition, Canonical Polyadic Decomposition, TensoRF with CP decomposition, TensoRF-CP

## Overview
CP Decomposition is a tensor factorization technique that represents a tensor as a sum of rank-one components, where each component is the outer product of vectors corresponding to each mode of the tensor [S1][S2, unofficial][S3, unofficial]. In the context of volume rendering, it is used to factorize the 3D (or 4D) radiance field tensor into compact vectors, significantly reducing the memory footprint compared to dense voxel grids [S1][S3, unofficial]. Unlike [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md), which uses a core tensor multiplied by factor matrices, CP uses only vectors, offering higher compactness but requiring more components to model complex scenes, leading to higher computational costs during reconstruction [S1][S3, unofficial].

## How it works
**Inputs and Outputs**
- Input: A tensor $\mathcal{T}$ of order $N$ (typically 3 or 4 in radiance fields) with dimensions $I \times J \times K$ (and potentially $P$ for appearance channels).
- Output: A set of factor vectors $\{\mathbf{v}_r^1\}_{r=1}^R, \{\mathbf{v}_r^2\}_{r=1}^R, \{\mathbf{v}_r^3\}_{r=1}^R$ for a 3D tensor, where $R$ is the rank (number of components).

**Data Structures**
The decomposition stores $N$ matrices of factor vectors. For a 3D tensor with dimensions $I, J, K$ and rank $R$:
- Mode 1 factors: Matrix $V^1 \in \mathbb{R}^{I \times R}$, where column $r$ is $\mathbf{v}_r^1$.
- Mode 2 factors: Matrix $V^2 \in \mathbb{R}^{J \times R}$, where column $r$ is $\mathbf{v}_r^2$.
- Mode 3 factors: Matrix $V^3 \in \mathbb{R}^{K \times R}$, where column $r$ is $\mathbf{v}_r^3$.
Each vector $\mathbf{v}_r^n$ contains $I, J,$ or $K$ scalar values respectively.

**Formulas**
The tensor $\mathcal{T}$ is approximated by the sum of $R$ rank-one outer products [S1][S2, unofficial][S3, unofficial]:
$$
\mathcal{T} = \sum_{r=1}^R \mathbf{v}_r^1 \circ \mathbf{v}_r^2 \circ \mathbf{v}_r^3
$$
where $\circ$ denotes the outer product. The element at indices $i, j, k$ is computed as [S1][S2, unofficial][S3, unofficial]:
$$
\mathcal{T}_{ijk} = \sum_{r=1}^R \mathbf{v}_{r,i}^1 \mathbf{v}_{r,j}^2 \mathbf{v}_{r,k}^3
$$
where $\mathbf{v}_{r,i}^1$ is the $i$-th element of the $r$-th vector in mode 1, and similarly for other modes.

For radiance fields, density ($\sigma$) and appearance features ($c$) are often factorized separately. If $P$ is the number of appearance channels, the total number of parameters for a grid with dimensions $I=J=K=n$ and total rank $R = R_\sigma + R_c$ is [S1]:
$$
\text{Params} = 3n(R_\sigma + R_c) + P R_c
$$
Here, $R_\sigma$ is the number of components for density and $R_c$ is the number of components for appearance. Typically, $R_\sigma < R_c$ when $R_\sigma > 8$ [S1].

**Steps for Reconstruction/Optimization**
1. **Initialization**: Initialize the factor matrices $V^1, V^2, V^3$ with random values or using a specific initialization scheme (e.g., SVD-based, though not explicitly detailed in sources, standard practice [G]).
2. **Forward Pass**: For a given ray, sample points in 3D space. For each point with indices $(i, j, k)$, compute the density and appearance features by evaluating the CP sum formula above.
3. **Rendering**: Use the computed density and appearance to render the pixel color via volume rendering integration.
4. **Loss Calculation**: Compute the loss between rendered and ground-truth images (e.g., L2 loss) [S1].
5. **Backpropagation**: Compute gradients of the loss with respect to the factor vectors $\mathbf{v}_r^n$.
6. **Update**: Update the factor vectors using an optimizer (e.g., Adam) [S1].
7. **Repeat**: Iterate until convergence.

**Parameters**
- $R$: The number of components (rank). Increasing $R$ increases expression capacity but also increases training time [S1].
- $R_\sigma, R_c$: Number of components for density and appearance. For TensoRF-CP, $R_1 = R_2 = R_3 = R$ for both density and appearance, assuming equal spatial complexity [S1].

**Edge Cases and Pitfalls**
- **High Compactness**: CP decomposition can require many components to model complex scenes due to its high compactness, leading to high computational costs in radiance field reconstruction [S1][S3, unofficial].
- **Degenerate Input**: If the tensor is not low-rank, CP may fail to converge or require a very large $R$, negating the memory benefits.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cp_forward(v1, v2, v3, i, j, k):
    """
    Compute the value of a 3D tensor element at indices (i, j, k)
    using CP decomposition factors.
    
    Args:
        v1: Matrix of shape (I, R), factor vectors for mode 1
        v2: Matrix of shape (J, R), factor vectors for mode 2
        v3: Matrix of shape (K, R), factor vectors for mode 3
        i: Index in mode 1
        j: Index in mode 2
        k: Index in mode 3
        
    Returns:
        Scalar value of the tensor at (i, j, k)
    """
    # Extract the r-th component vectors for the given indices
    # v1[i, :] is the i-th row of v1, shape (R,)
    # v2[j, :] is the j-th row of v2, shape (R,)
    # v3[k, :] is the k-th row of v3, shape (R,)
    
    component_values = v1[i, :] * v2[j, :] * v3[k, :]
    
    # Sum over all R components
    return sum(component_values)

def cp_reconstruct(v1, v2, v3, indices):
    """
    Reconstruct values for multiple points.
    
    Args:
        v1: Matrix of shape (I, R)
        v2: Matrix of shape (J, R)
        v3: Matrix of shape (K, R)
        indices: List of tuples (i, j, k)
        
    Returns:
        List of scalar values
    """
    results = []
    for i, j, k in indices:
        val = cp_forward(v1, v2, v3, i, j, k)
        results.append(val)
    return results
```

## Subtopics

### [Vector-Matrix (VM) Decomposition](3452-vector-matrix-vm-decomposition.md)
Vector-Matrix (VM) decomposition is a tensor factorization technique that relaxes the low-rank constraints for two modes of a tensor by factorizing it into compact vector and matrix factors, effectively acting as a special case of Block Term Decomposition (BTD). Unlike [CP Decomposition](2255-cp-decomposition.md), which uses a sum of outer products of pure vectors, VM uses a sum of vector-matrix outer products. This allows each component to express more complex tensor data of higher ranks, significantly reducing the required number of components compared to CP while maintaining a memory complexity of $O(N^2)$ compared to the $O(N^3)$ of dense grids.

## Operating characteristics
- **Time Complexity**: Training time increases with the number of components $R$ [S1]. Reconstruction speed is generally faster than dense grids but slower than VM decomposition for the same quality [S1].
- **Memory Complexity**: Space complexity is reduced from $O(n^3)$ to $O(n)$ compared to direct voxel grid optimization, where $n$ is the grid resolution dimension [S1][S3, unofficial]. Specifically, for a grid of size $n \times n \times n$ and rank $R$, the memory is $O(nR)$.
- **Measured Performance**: For a 300x300x300 feature grid with P=27 channels, TensoRF-CP with R=192 uses 360K parameters, achieving 0.048% compression rate compared to 756M parameters in a dense grid [S1]. TensoRF with CP decomposition achieves reconstruction in less than 30 minutes with a model size smaller than 4 MB [S3, unofficial]. On the Synthetic-NeRF dataset, CP-384 achieves PSNR 31.56/31.23 (w/ L1 / w/o L1), SSIM 0.949/0.947, LPIPS 0.076/0.078 [S1].
- **Strengths**: Significant memory reduction, fast reconstruction compared to MLP-based NeRF [S3, unofficial].
- **Limitations**: Requires many components for complex scenes, leading to high computational costs [S1][S3, unofficial]. Less expressive than VM decomposition for the same number of components [S1].
- **When to Use**: When memory is constrained and scenes are not overly complex, or as a baseline for tensorial radiance fields.
- **When Not to Use**: When high-quality reconstruction of complex scenes is required and computational resources are limited, VM decomposition is preferred [S1].

## Background
CP Decomposition, also known as CANDECOMP/PARAFAC, is a classical tensor decomposition method. In the context of volume rendering, it was introduced in the TensoRF paper (Chen et al., 2022) as a method to factorize radiance fields into compact vectors [S1][S3, unofficial]. It builds on the idea of using tensor factorization for efficient scene representation, offering a low-rank regularization in the optimization process [S1].

## Sources
[S1] TensoRF: Tensorial Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_20, fetched 2026-10-26
[S2] PDF TensoRF: Tensorial Radiance Fields - ECVA, https://www.ecva.net/papers/eccv_2022/papers_ECCV/papers/136920332.pdf, fetched 2026-10-29 (unofficial)
[S3] Tensorial Radiance Fields - TensoRF, https://www.cvlibs.net/publications/Chen2022ECCV.pdf, fetched 2026-10-29 (unofficial)
