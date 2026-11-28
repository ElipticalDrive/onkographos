# Discrete Distribution Sampler

Also called: Discrete Distribution Sampling, DistributionSampler

## Overview
The Discrete Distribution Sampler selects indices from a finite set of weighted choices or a probability density function tensor using a uniform random number [S1, S2, unofficial]. It differs from continuous interpolation methods by operating directly on discrete probability mass functions, supporting both stochastic sampling and deterministic top-k selection [S1, S2, unofficial].

## How it works
The method supports two primary modes: stochastic sampling for unbiased estimation and deterministic top-k selection for stable training or debugging [S2, unofficial]. The input consists of weighted choices or a probability density function tensor, along with a uniform random variable $u \in [0, 1)$ [S1, S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `choices`: A list of items, each containing an identifier and a float weight $w_i \ge 0$ [S1, unofficial].
    *   `pdf`: A float tensor of shape `Float[*batch, bucket]` representing discrete bin weights [S2, unofficial].
    *   `u`: A uniform random number in $[0, 1)$ [S1, unofficial].
    *   `deterministic`: A boolean flag. If `true`, selects top-k indices; if `false`, samples stochastically [S2, unofficial].
    *   `num_samples`: The number of indices to select (used in deterministic mode) [S2, unofficial].
*   **Outputs**:
    *   `index`: The selected index or indices [S1, S2, unofficial].
    *   `densities`: The probability density values at the selected indices [S2, unofficial].

### Data Structures
*   **Weighted Choice**: A structure containing an identifier and a float weight [S1, unofficial].
*   **PDF Tensor**: Input tensor of shape `Float[*batch, bucket]`, where `bucket` represents the number of discrete bins [S2, unofficial].
*   **Index Tensor**: Output tensor of shape `Int64[*batch, sample]` [S2, unofficial].

### Algorithm Steps

#### Stochastic Sampling Mode
1.  **Calculate Total Weight**: Compute the sum of all weights: $W_{total} = \sum_{i} w_i$ [S1, unofficial].
2.  **Scale Random Number**: Map the uniform random variable $u$ to the range $[0, W_{total})$: $u' = u \times W_{total}$ [S1, unofficial].
3.  **Iterative Subtraction**: Iterate through the choices in order. For each choice $i$ with weight $w_i$: $u' \leftarrow u' - w_i$. If $u' \le 0$, return the current choice's index [S1, unofficial].
4.  **Fallback**: If the loop completes without returning (due to floating-point precision issues where $u'$ remains slightly positive), return the last element's index [S1, unofficial].

#### Deterministic (Top-K) Mode
1.  **Select Indices**: Sort the probability densities (weights) in descending order and select the top $k$ indices [S2, unofficial].
2.  **Gather Values**: Retrieve the corresponding values from the target tensor using the selected indices [S2, unofficial].

### Pseudocode (Stochastic)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_discrete_distribution_stochastic(choices, u):
    """
    Samples a single index from a list of weighted choices using inverse CDF sampling.
    
    Args:
        choices: List of objects with attribute 'weight' (float >= 0).
        u: Uniform random number in [0, 1).
        
    Returns:
        index: The index of the selected choice.
    """
    # Step 1: Calculate total weight
    total_w = sum(choice.weight for choice in choices)
    
    # Step 2: Scale u
    u_scaled = u * total_w
    
    # Step 3: Iterate and subtract
    for i, choice in enumerate(choices):
        u_scaled -= choice.weight
        if u_scaled <= 0.0:
            return i
            
    # Step 4: Fallback
    return len(choices) - 1
```

### Pseudocode (Tensor-based Gather)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gather_discrete_values(target, index, bucket_dim):
    """
    Gathers values from a target tensor using an index tensor, handling broadcasting.
    
    Args:
        target: Tensor of shape [..., bucket_size, ...].
        index: Int64 tensor of shape [..., sample_size].
        bucket_dim: The dimension in target corresponding to the buckets.
        
    Returns:
        gathered: Tensor with values selected by index.
    """
    # Step 1: Determine bucket dimension as second-to-last dimension of index
    bucket_dim = len(index.shape) - 1
    
    # Step 2: Expand index dimensions to match target dimensionality
    while len(index.shape) < len(target.shape):
        index = index + (None,)
        
    # Step 3: Broadcast index to match target shape, preserving bucket dimension size
    # (Standard tensor broadcasting logic applied here)
    
    # Step 4: If target's bucket dimension size is 1, reset index to zeros
    if target.shape[bucket_dim] == 1:
        index = [[0] * len(index[0])] * len(index)
        
    # Step 5: Perform gather operation along bucket_dim
    # (Standard gather operation along the specified dimension)
    return target.gather(dim=bucket_dim, index=index)
```

## Subtopics

### [Piecewise-Constant PDF Sampling](12184-piecewise-constant-pdf-sampling.md)
A variant that generates sample coordinates from a discrete probability distribution defined by bin endpoints and logits by inverting the cumulative distribution function (CDF). It differs from sibling methods by explicitly constructing a step-wise CDF from softmax-normalized weights and using inverse interpolation to map uniform random variables to bin centers or intervals, ensuring samples respect the learned probability mass of each bin.

### [Multinomial Sampling Fallback](16543-multinomial-sampling-fallback.md)
A discrete distribution sampling method that selects indices from a weight distribution without replacement, designed to handle large-scale tensors by switching between backends. It addresses the memory and performance limitations of native sampling on very large weight arrays by falling back to an alternative implementation when the tensor size exceeds a specific threshold, ensuring robustness in 3D Gaussian Splatting and similar rendering pipelines.

### [PMF Sampling](16828-pmf-sampling.md)
Draws indices from a discrete probability mass function (PMF) with replacement, using inverse transform sampling via the cumulative distribution function (CDF). It differs from its sibling [Multinomial Sampling Fallback](16543-multinomial-sampling-fallback.md) by explicitly constructing the CDF and using search operations to map uniform random variables to indices, rather than relying on library-specific multinomial primitives or fallback logic.

## Operating characteristics
*   **Time Complexity**: Stochastic sampling requires a linear scan over the weights [S1, unofficial]. Deterministic mode uses top-k selection [S2, unofficial].
*   **Memory Complexity**: Stores the weight tensor and output index tensor [S1, S2, unofficial].
*   **Precision**: The iterative subtraction method may encounter floating-point precision issues where the loop completes without $u \le 0$, requiring a fallback to the last element [S1, unofficial].
*   **Strengths**: Supports both stochastic and deterministic sampling modes [S1, S2, unofficial].
*   **Limitations**: Not given in the sources.

## Background
The Discrete Distribution Sampler is used in modern volume rendering and 3D Gaussian splatting pipelines to select sample positions based on per-bin weights [S1, S2, unofficial]. The specific implementations cited are found in contemporary rendering codebases [S1, S2, unofficial].

## Sources
[S1] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
[S2] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
