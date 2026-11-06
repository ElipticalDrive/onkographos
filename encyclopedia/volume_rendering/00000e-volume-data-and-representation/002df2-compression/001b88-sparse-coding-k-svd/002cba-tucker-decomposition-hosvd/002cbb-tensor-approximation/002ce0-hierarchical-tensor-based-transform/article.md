# Hierarchical Tensor-based Transform

Also called: Wu hierarchical tensor, Wu et al. hierarchical tensor

## Overview
The Hierarchical Tensor-based Transform is a compression scheme for volume data that recursively partitions the input domain into smaller blocks, defining a truncated tensor-product basis for each block [S1][S2]. It represents the original volume tensor as a summation of incomplete tensor approximations across multiple resolution levels, where tensors at each level are subdivided into residual tensors passed from higher levels [S1][S2].

## How it works
**Inputs:** A 3D volume tensor $V$.
**Outputs:** Hierarchical coefficients comprising truncated tensor-product bases and residual tensors for each partition level [S1][S2].

**Data Structures:**
*   **Blocks:** Recursive subdivisions of the volume domain [S1][S2].
*   **Bases:** A truncated tensor-product basis defined for each block [S1][S2].
*   **Residuals:** Residual tensors passed from higher levels to lower levels [S1][S2].

**Steps:**
1.  **Recursive Partitioning:** The entire domain of the input volume is recursively partitioned into smaller blocks [S1][S2].
2.  **Basis Definition:** For each block in the hierarchy, a truncated tensor-product basis is defined [S1][S2].
3.  **Residual Subdivision:** Tensors at each level are subdivided into residual tensors that are passed from the higher levels [S1][S2].
4.  **Approximation Summation:** The original tensor is represented as a summation of incomplete tensor approximations at multiple levels [S1][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_tensor_transform(volume_tensor, max_depth):
    """
    Implements the Hierarchical Tensor-based Transform.
    """
    # Step 1: Recursive partitioning of the domain into blocks
    blocks = []
    def partition_recursive(tensor, depth):
        if depth >= max_depth:
            blocks.append(tensor)
            return
        # Standard subdivision: split each dimension in half
        sub_tensors = split_tensor(tensor)
        partition_recursive(sub_tensors[0], depth + 1)
        partition_recursive(sub_tensors[1], depth + 1)
    partition_recursive(volume_tensor, 0)

    # Step 2: Define truncated tensor-product basis for each block
    bases = {}
    for i, block in enumerate(blocks):
        # The source mentions "truncated tensor-product basis" but does not specify
        # the truncation criterion or computation method (e.g., SVD rank threshold) [G].
        bases[i] = block  # Basis computation step omitted per sources [G]

    # Step 3: Subdivide tensors into residual tensors passed from higher levels
    residuals = {}
    # Assuming a hierarchical level structure
    for level in range(max_depth, -1, -1):
        for block in blocks_at_level(level):
            # Get residual from parent (higher level)
            parent_residual = get_parent_residual(block)
            # Subdivide current tensor into residual for children
            # The exact mathematical operation for subdivision is not given [G].
            current_residual = subdivide_tensor_into_residual(block, parent_residual)
            residuals[block.id] = current_residual

    # Step 4: Represent original tensor as summation of incomplete approximations
    # The original tensor V is approximated by summing contributions from all levels.
    # V_approx = sum(incomplete_approximations)
    # The source does not provide the explicit formula for the summation or how
    # the bases and residuals combine to form the approximation [G].
    approximation = compute_summation_of_incomplete_approximations(bases, residuals)

    return approximation
```

**Parameters:**
*   **Partition Depth:** The number of recursive subdivisions. Not given in the sources.
*   **Truncation Rank:** The rank of the tensor-product basis for each block. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Basis Computation:** The sources do not specify how the "truncated tensor-product basis" is computed (e.g., via HOSVD, CP decomposition, or other factorization methods). This is a critical implementation gap [G].
*   **Residual Propagation:** The exact mechanism for passing residuals from higher to lower levels is not detailed [G].
*   **Incomplete Approximations:** The term "incomplete tensor approximations" is used, but the specific mathematical form of these approximations is not provided [G].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.

## Background
The Hierarchical Tensor-based Transform was proposed by Wu et al. in 2008 [S1][S2]. It is cited in the context of dictionary-based methods and tensor approximation techniques for compressed GPU-based direct volume rendering [S1][S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
