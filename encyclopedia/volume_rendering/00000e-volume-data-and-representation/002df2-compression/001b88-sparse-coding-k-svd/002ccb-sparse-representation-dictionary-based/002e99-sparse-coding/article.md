# Sparse Coding
Also called: Sparse Coding Decompression, Sparse coding of voxel blocks

## Overview
Sparse coding decompresses volume data by reconstructing voxel blocks as linear combinations of codewords from a dictionary [S1][S2]. It distinguishes itself from its parent method [Sparse Representation (Dictionary Based)](5366-sparse-representation-dictionary-based.md) by explicitly utilizing a linear combination of multiple codewords rather than a single best match, and from its sibling [Sparse Representation Modeling](5392-sparse-representation-modeling.md) by focusing on the decomposition and reconstruction mechanics of voxel blocks rather than statistical coefficient modeling [G].

## How it works
**Inputs and Outputs:**
- Input: Compressed data consisting of indices pointing to codewords in a dictionary and their associated weighting coefficients (sparse representation) for each voxel block [S1][S2].
- Output: Reconstructed 3D volume data.

**Data Structures:**
- Dictionary $D$: A matrix where columns are codewords.
- Sparse Representation: For each block, a set of $S$ indices into the dictionary and $S$ corresponding scalar weights [S1][S2].

**Steps:**
1. For each voxel block in the volume:
   a. Access the $S$ indices identifying the relevant codewords from the dictionary [S1][S2].
   b. Access the $S$ weighting coefficients associated with those indices [S1][S2].
   c. Compute the reconstructed voxel values by calculating the linear combination of the selected codewords weighted by their coefficients [S1][S2].
   d. Store or stream the reconstructed voxels.

**Formulas:**
The reconstruction of a voxel block $V$ is given by:
$$ V = \sum_{i=1}^{S} w_i \cdot d_i $$
Where:
- $S$ is the sparsity (number of non-zero coefficients) [S1][S2].
- $w_i$ is the weighting coefficient for the $i$-th codeword [S1][S2].
- $d_i$ is the $i$-th codeword from the dictionary $D$ [S1][S2].

**Parameters:**
- $N$: Number of blocks [S1][S2].
- $S$: Sparsity (number of codewords used per block) [S1][S2].
- $M$: Number of dictionaries (used for comparison with HVQ) [S1][S2].

**Edge Cases and Pitfalls:**
- Precision: The method relies on floating-point arithmetic for the linear combination. Standard floating-point precision should be used for weights and codewords [G].
- Boundary: Not given in the sources.
- Degenerate input: If $S=0$, the block reconstructs to zero [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_sparse_coding(volume_blocks, dictionary, sparse_data):
    """
    Reconstructs volume data from sparse coding representation.
    
    Args:
        volume_blocks: List of block objects to be filled.
        dictionary: List of codeword vectors.
        sparse_data: List of tuples (indices, weights) for each block.
                     indices: list of S integers pointing to dictionary.
                     weights: list of S floats.
    """
    for block_idx, block in enumerate(volume_blocks):
        indices, weights = sparse_data[block_idx]
        S = len(indices)
        
        # Initialize block voxels to zero
        block.voxels = [0.0] * block.size
        
        # Linear combination of codewords
        for i in range(S):
            codeword = dictionary[indices[i]]
            weight = weights[i]
            
            # Add weighted codeword contribution to block voxels
            for v_idx in range(block.size):
                block.voxels[v_idx] += weight * codeword[v_idx]
```

## Subtopics

### Sparse Representation Modeling
Focuses on the statistical modeling of sparse coefficients rather than just the decomposition mechanics [G]. Choose when the goal is to model the distribution of sparse codes for better compression or denoising. [Link](5392-sparse-representation-modeling.md)

## Operating characteristics
- **Time Complexity**: $O(N \times S)$, where $N$ is the number of blocks and $S$ is the sparsity [S1][S2]. This is linear with respect to the number of blocks and sparsity [S1][S2].
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: The computational complexity is very similar to Hierarchical Vector Quantization (HVQ) [S1][S2].
- **Scaling**: Scales linearly with the number of blocks and the sparsity level [S1][S2].
- **Strengths**: Allows parallel decompression of voxels by accessing codewords and coefficients [S1][S2].
- **Limitations**: Requires storage of multiple coefficients and indices per block compared to VQ [G].
- **When to use**: When parallel decompression is desired and low computational complexity $O(N \times S)$ is required [S1][S2].
- **When not**: When minimal computational overhead is critical and VQ suffices, or when extremely high compression is needed and dictionary learning overhead is prohibitive [G].

## Background
Sparse coding in the context of compressed GPU-based direct volume rendering was discussed in relation to Vector Quantization (VQ) and Hierarchical VQ (HVQ) [S1][S2]. The method leverages the idea that voxel blocks can be represented as a linear combination of dictionary codewords, maintaining low computational complexity during decompression [S1][S2]. It builds on earlier work in compressed volume rendering, such as that by Gobbetti et al. [GIM12], which proposed parallel access to codewords and coefficients [S1][S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
