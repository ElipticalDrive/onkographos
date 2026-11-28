# Tucker Thresholding

Also called: Core Tensor Thresholding, tthresh

## Overview
Tucker Thresholding is a post-processing compression technique applied to data already represented via Tucker decomposition. It achieves compression by zeroing out small-magnitude elements in the core tensor, followed by run-length and entropy encoding of the resulting sparse structure [S2, unofficial]. It is distinguished from its sibling [Truncated Tensor Decomposition](7540-truncated-tensor-decomposition.md) by operating on the values within the core rather than reducing the core's dimensions, and from [Adaptive Rank Selection via Binary Search](9822-adaptive-rank-selection-via-binary-search.md) by requiring explicit decompression for access or rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A Tucker-decomposed volume represented by a core tensor $\mathcal{G} \in \mathbb{R}^{R_1 \times R_2 \times \dots \times R_N}$ and factor matrices $U^{(n)} \in \mathbb{R}^{I_n \times R_n}$ for $n=1 \dots N$.
*   **Output:** A compressed bitstream containing the thresholded core tensor (encoded via run-length and entropy coding) and the factor matrices.

**Data Structures:**
The core tensor $\mathcal{G}$ is stored as a dense array of size $R_1 \times R_2 \times \dots \times R_n$ before thresholding. After thresholding, it becomes sparse. The factor matrices $U^{(n)}$ are stored in standard dense format.

**Parameters:**
*   **Threshold ($\tau$):** A scalar value determining which elements are considered negligible. The sources do not specify the exact selection criterion for $\tau$ (e.g., fixed value, percentile, or error-bound optimization) [S2, unofficial].

**Steps:**
1.  **Quantization:** The core tensor elements are quantized. The sources indicate that coefficients are roughly logarithmic, and logarithmic quantization works best, typically using 8 bits plus 1 sign bit [S2, unofficial].
2.  **Thresholding:** Iterate through all elements $g_{i_1, i_2, \dots, i_N}$ of the core tensor $\mathcal{G}$. If $|g_{i_1, i_2, \dots, i_N}| < \tau$, set the element to zero.
3.  **Encoding:** Apply run-length encoding to the sequence of zeroed elements, followed by entropy encoding (e.g., Huffman or Arithmetic coding) to compress the non-zero values and run-length codes [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tucker_thresholding(core_tensor, factor_matrices, threshold):
    """
    Compresses a Tucker-decomposed volume by thresholding the core tensor.
    
    Args:
        core_tensor: 3D (or N-D) array of shape (R1, R2, ..., RN).
        factor_matrices: List of N arrays U^(n) of shape (In, Rn).
        threshold: Float scalar. Elements with absolute value below this are zeroed.
        
    Returns:
        compressed_core: Encoded data for the thresholded core.
        factor_matrices: The original factor matrices (unchanged).
    """
    # Step 1: Thresholding
    # Create a mask for elements below the threshold
    mask = abs(core_tensor) < threshold
    
    # Zero out small elements
    core_tensor[mask] = 0.0
    
    # Step 2: Encoding
    # Flatten the core tensor to a 1D array for run-length encoding
    flat_core = core_tensor.flatten()
    
    # Run-length encoding
    # Identify runs of zeros and non-zeros
    # Note: The specific entropy coding algorithm (e.g., Huffman) is not detailed in sources [S2, unofficial]
    compressed_core = run_length_and_entropy_encode(flat_core)
    
    return compressed_core, factor_matrices

def run_length_and_entropy_encode(data):
    """
    Placeholder for the specific run-length and entropy encoding implementation.
    Sources state "Run-length + Entropy encoding" but do not provide the algorithm [S2, unofficial].
    """
    # Implementation details not given in the sources.
    pass
```

**Edge Cases and Pitfalls:**
*   **Explicit Decompression:** Accessing or rendering the data requires explicit decompression, which impacts interactive performance [S1].
*   **Precision:** Logarithmic quantization is recommended to handle the wide dynamic range of coefficients, which are roughly logarithmic in distribution [S2, unofficial].

## Subtopics

### Tucker Truncation
Reduces the ranks of the core tensor and factor matrices during visualization to achieve interactive level-of-detail. It is very fast to apply but selects different features based on rank [S2, unofficial]. See [Truncated Tensor Decomposition](7540-truncated-tensor-decomposition.md).

### Adaptive Rank Selection via Binary Search
Uses binary search to find optimal ranks for interactive rendering. See [Adaptive Rank Selection via Binary Search](9822-adaptive-rank-selection-via-binary-search.md).

### Rank-Residual Learning
A method involving learning residuals for rank reduction. See [Rank-Residual Learning](13305-rank-residual-learning.md).

### Tensor Rank Reduction
General techniques for reducing tensor ranks. See [Tensor Rank Reduction](15445-tensor-rank-reduction.md).

## Operating characteristics
*   **Compression Ratio:** Tucker Thresholding achieves significantly better compression ratios than wavelet-based methods (DCT, Haar, Daubechies, Coiflets, Biorthogonal) for the same compression factor on datasets like Bonsai, Foot, Skull, and Wood [S2, unofficial].
*   **Relative Error:** It generally achieves lower relative error compared to wavelet transforms at equivalent compression factors [S2, unofficial].
*   **Performance:** It takes significantly longer to compress datasets compared to other methods [S1].
*   **Hardware:** Benchmarks were performed on a dual Intel Xeon E5-2699 (88-core) server with 256GB main memory [S1].
*   **Strengths:** High compression ratio and low relative error for suitable data distributions.
*   **Limitations:** Slow compression time and the need for explicit decompression for access/rendering [S1].

## Background
Tucker Thresholding is part of the broader field of tensor approximation for volume data compression. It was compared against neural network-based compression methods (like neurcomp) and wavelet transforms in studies involving datasets such as PigHeart, 10atmHR, Bonsai, Foot, and Skull [S1][S2, unofficial]. The technique leverages the sparsity induced in the core tensor of a Tucker decomposition to enable efficient entropy coding [S2, unofficial].

## Sources
[S1] Interactive Volume Visualization via Multi-Resolution Hash Encoding ..., https://arxiv.org/abs/2207.11620, fetched 2026-10-26
[S2] PDF Analysis of Tensor Approximation for Compression-Domain Volume ..., https://www.ifi.uzh.ch/vmml/publications/analysistensorapproximationtalk/tensorapproximation.pdf, fetched 2026-10-29 (unofficial)
