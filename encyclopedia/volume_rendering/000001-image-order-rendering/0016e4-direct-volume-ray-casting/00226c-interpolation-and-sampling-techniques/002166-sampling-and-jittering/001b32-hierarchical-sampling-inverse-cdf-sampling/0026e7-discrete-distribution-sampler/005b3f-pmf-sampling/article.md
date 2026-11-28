# PMF Sampling

Also called: Discrete Distribution Sampler with Replacement, Inverse CDF Sampling for Discrete PMF, Weighted Subset Random Sampler

## Overview
PMF Sampling draws indices from a discrete probability mass function (PMF) with replacement, using inverse transform sampling via the cumulative distribution function (CDF). It differs from its sibling [Multinomial Sampling Fallback](16543-multinomial-sampling-fallback.md) by explicitly constructing the CDF and using search operations to map uniform random variables to indices, rather than relying on library-specific multinomial primitives or fallback logic. It is chosen when deterministic CDF-based mapping is required for gradient flow or reproducibility in stochastic sampling pipelines.

## How it works
**Inputs:**
- `pmf`: A tensor of shape `[batch_size, *shape, n_samples, 1]` representing the probability mass function. The sum along dimension `-2` (the `n_samples` dimension) must equal 1.0.
- `n_samples`: An integer specifying the number of samples to draw. If 0, the number of samples defaults to the length of the support dimension (`n_samples` in the input tensor).

**Outputs:**
- `indices`: A tensor of sampled indices, reshaped to match the original batch and shape dimensions, with the last dimension being the number of drawn samples.

**Data Structures:**
- `cdf`: A tensor of the same shape as `pmf`, where each element is the cumulative sum of `pmf` along the support dimension.

**Steps:**
1. **Filter Weights:** If the sampler is initialized with a subset of indices, filter the `pmf` weights to match these indices. [S2, unofficial]
2. **Normalize:** Ensure the `pmf` sums to 1 along the support dimension. [S1, unofficial]
3. **Compute CDF:** Calculate the cumulative distribution function along the support dimension (dim=-2).
   $$ \text{CDF}_i = \sum_{j=0}^{i} \text{PMF}_j $$
   [S1, unofficial]
4. **Determine Sample Count:** If `n_samples` is 0, set it to the size of the support dimension. Otherwise, use the provided `n_samples`. [S2, unofficial]
5. **Generate Randoms:** Generate `n_samples` uniform random numbers $u \in [0, 1)$ for each sample. [S1, unofficial]
6. **Search Sorted:** Use `searchsorted` to find the index $k$ in the CDF such that $\text{CDF}_{k-1} < u \leq \text{CDF}_k$. This maps the uniform random value to the corresponding index in the support. [S1, unofficial]
7. **Clamp and Reshape:** Clamp the resulting indices to the valid range $[0, \text{support\_size} - 1]$ to handle edge cases (e.g., $u=1.0$ due to floating point precision). Reshape the indices to the original batch and shape dimensions. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def pmf_sampling(pmf: torch.Tensor, n_samples: int) -> torch.Tensor:
    """
    Samples indices from a discrete PMF with replacement using inverse CDF.
    
    Args:
        pmf: Tensor of shape [batch_size, *shape, n_support, 1]. Sum along dim=-2 must be 1.
        n_samples: Number of samples to draw. If 0, defaults to n_support.
        
    Returns:
        indices: Tensor of sampled indices.
    """
    # 1. Ensure pmf sums to 1 along the support dimension (dim=-2)
    # [G] Add small epsilon for numerical stability if needed, though source assumes valid input
    pmf_sum = pmf.sum(dim=-2, keepdim=True)
    pmf = pmf / pmf_sum
    
    # 2. Compute CDF along the support dimension
    cdf = torch.cumsum(pmf, dim=-2)
    
    # 3. Determine number of samples
    support_size = pmf.shape[-2]
    if n_samples == 0:
        n_samples = support_size
        
    # 4. Generate uniform random numbers
    # Shape: [batch_size, *shape, n_samples, 1]
    # [G] Assuming batch and shape dimensions are preserved for random generation
    batch_shape = pmf.shape[:-2]
    u = torch.rand((*batch_shape, n_samples, 1), device=pmf.device)
    
    # 5. Searchsorted to find indices in CDF
    # searchsorted returns the index where u would be inserted to maintain order
    # This effectively finds the smallest k such that CDF[k] >= u
    indices = torch.searchsorted(cdf.squeeze(-1), u.squeeze(-1), side='right')
    
    # 6. Clamp indices to valid range [0, support_size - 1]
    indices = torch.clamp(indices, 0, support_size - 1)
    
    # 7. Reshape to original batch/shape dimensions with n_samples
    # The output shape should be [batch_size, *shape, n_samples]
    return indices
```

**Parameters:**
- `pmf`: Input probability mass tensor. Shape `[batch_size, *shape, n_samples, 1]`. Sum along dim=-2 must be 1. [S1, unofficial]
- `n_samples`: Number of samples to draw. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Summation Error:** If the PMF does not sum to 1, the CDF will not reach 1.0, leading to incorrect sampling. Normalization is required. [S1, unofficial]
- **Boundary Conditions:** Uniform random numbers can be exactly 1.0 due to floating-point representation. Clamping indices to `support_size - 1` prevents out-of-bounds access. [S1, unofficial]
- **Zero Weights:** If some weights are zero, the CDF will have flat regions. `searchsorted` will return the first index in the flat region, which is a valid deterministic choice.

## Operating characteristics
- **Time Complexity:** $O(N \log M)$ where $N$ is the number of samples and $M$ is the support size, due to `searchsorted`. CDF computation is $O(M)$.
- **Memory Complexity:** $O(M)$ for storing the CDF.
- **Strengths:** Deterministic mapping from random seed to index (given fixed randoms), compatible with gradient-based optimization if using differentiable search approximations (though standard `searchsorted` is not differentiable).
- **Limitations:** Not differentiable through the sampling step. Slower than library-specific multinomial samplers for large $M$ if $N$ is small.
- **When to use:** When inverse transform sampling is explicitly required, or when integrating with systems that expect CDF-based sampling.
- **When not to use:** When performance is critical and $M$ is very large, or when differentiability through the sampling step is required (consider Gumbel-Softmax or straight-through estimators).

## Background
PMF Sampling is a standard technique in stochastic simulation and computer graphics for sampling from discrete distributions. It is implemented in various 3D generation frameworks, such as `gsgen3d` and `guassian-diffusion`, often as part of a larger `Discrete Distribution Sampler` class. [S1, S2, unofficial]

## Sources
[S1] gsgen3d/gsgen, https://github.com/gsgen3d/gsgen, fetched 2026-10-27 (unofficial)
[S2] pbonazzi/guassian-diffusion, https://github.com/pbonazzi/guassian-diffusion, fetched 2026-10-28 (unofficial)
