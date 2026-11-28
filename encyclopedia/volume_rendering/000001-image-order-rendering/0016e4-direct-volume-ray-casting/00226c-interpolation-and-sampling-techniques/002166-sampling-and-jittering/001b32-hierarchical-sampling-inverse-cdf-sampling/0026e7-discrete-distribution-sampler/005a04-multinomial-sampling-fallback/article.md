# Multinomial Sampling Fallback

Also called: Adaptive Multinomial Sample, MultinomialSampler

## Overview
Multinomial Sampling Fallback is a discrete distribution sampling method that selects indices from a weight distribution without replacement, designed to handle large-scale tensors by switching between PyTorch and NumPy backends. It addresses the memory and performance limitations of native PyTorch sampling on very large weight arrays by falling back to NumPy when the tensor size exceeds a specific threshold, ensuring robustness in 3D Gaussian Splatting and similar rendering pipelines [S1, S2, unofficial].

## How it works
The method operates on a set of weights associated with discrete elements, selecting a specified number of unique indices. It validates inputs, normalizes weights, and delegates the sampling operation to either PyTorch or NumPy based on the size of the input data.

**Inputs and Outputs:**
- **Input:** `weights` (1D tensor of non-negative weights), `n` (integer, number of samples to draw), `replacement` (boolean, default `True` in general definition, but `False` in specific implementation context for unique selection).
- **Output:** A tensor of selected indices.

**Data Structures:**
- `weights`: A 1D PyTorch tensor. If using the NumPy fallback, it is converted to a NumPy array.
- `indices`: The list or tensor of indices from which weights are drawn. In the specific implementation [S2, unofficial], weights are extracted corresponding to provided indices.

**Steps:**
1. **Validate `n`:** Check if `n` is a non-negative integer. If `n` is 0, set `n` to the length of the weights [S2, unofficial].
2. **Extract Weights:** If indices are provided, extract the corresponding weights and convert them to a PyTorch tensor of type `double` (float64) [S2, unofficial].
3. **Check Size Threshold:** Determine the number of elements (`num_elements`) in the weight distribution.
4. **Branch Execution:**
   - **If `num_elements` > $2^{24}$:**
     - Normalize the weights so they sum to 1 [S1, unofficial].
     - Convert the weights to a NumPy array [S1, unofficial].
     - Sample indices using `np.random.choice` with the normalized weights and `replace=False` (for unique sampling) [S1, S2, unofficial].
     - Convert the resulting indices back to a PyTorch tensor [S1, unofficial].
     - Move the tensor to the original device [S1, unofficial].
   - **Else:**
     - Use `torch.multinomial` with the weights, `n`, and `replacement=False` [S2, unofficial].
     - Yield the corresponding indices [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def multinomial_sampling_fallback(weights, n, replacement=False):
    """
    Selects n indices from weights using multinomial sampling.
    Falls back to NumPy for large tensors to avoid PyTorch limitations.
    
    Args:
        weights: 1D Tensor of weights.
        n: Integer, number of samples to draw.
        replacement: Boolean, whether to sample with replacement.
        
    Returns:
        Tensor of selected indices.
    """
    # Validate n
    if not isinstance(n, int) or n < 0:
        raise ValueError("n must be a non-negative integer")
    
    # If n is 0, sample all elements
    if n == 0:
        n = len(weights)
        
    # Ensure weights are double precision for numerical stability
    weights = weights.double()
    
    # Threshold for fallback
    THRESHOLD = 2**24
    
    if len(weights) > THRESHOLD:
        # Fallback to NumPy
        # Normalize weights
        sum_weights = torch.sum(weights)
        if sum_weights == 0:
            # Handle degenerate case: uniform distribution if all weights are zero
            probs = torch.ones_like(weights) / len(weights)
        else:
            probs = weights / sum_weights
            
        # Convert to NumPy
        probs_np = probs.cpu().numpy()
        
        # Sample using NumPy
        # Note: np.random.choice does not support direct probability array in older versions,
        # but modern versions do. We assume standard behavior.
        indices_np = np.random.choice(len(probs_np), size=n, replace=replacement, p=probs_np)
        
        # Convert back to Tensor and move to original device
        indices = torch.from_numpy(indices_np).to(weights.device)
        return indices
    else:
        # Native PyTorch sampling
        indices = torch.multinomial(weights, num_samples=n, replacement=replacement)
        return indices
```

**Parameters:**
- `weights`: 1D Tensor of weights for each element [S1, unofficial].
- `n`: Integer, number of samples to draw [S1, unofficial].
- `replacement`: Boolean, whether to sample with replacement. Default `True` in general definition, but `False` in the specific implementation [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Weights:** If all weights are zero, normalization fails. The implementation should handle this by defaulting to a uniform distribution [G].
- **Device Mismatch:** The NumPy fallback requires moving data to CPU, which can be slow. Ensure the final tensor is moved back to the original device [S1, unofficial].
- **Precision:** Weights are converted to `double` to maintain precision during normalization and sampling [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for normalization and conversion, where $N$ is the number of elements. Sampling is $O(n)$ for $n$ samples.
- **Memory Complexity:** $O(N)$ for storing weights and intermediate arrays.
- **Performance:** The fallback is triggered for $N > 2^{24}$, avoiding potential memory or performance issues in PyTorch's `multinomial` for very large tensors [S1, unofficial].
- **Strengths:** Robustness for large-scale data, compatibility with both PyTorch and NumPy ecosystems.
- **Limitations:** Device transfer overhead when using NumPy fallback. Not differentiable w.r.t. camera intrinsics in some contexts [S1, unofficial].

## Background
The method is used in 3D Gaussian Splatting implementations, such as `splatad` and `guassian-diffusion`, to handle aliasing and efficient sampling of Gaussian primitives [S1, S2, unofficial]. It is part of a broader effort to optimize rendering pipelines for large-scale 3D scenes.

## Sources
[S1] carlinds/splatad, https://github.com/carlinds/splatad, fetched 2026-10-27 (unofficial)
[S2] pbonazzi/guassian-diffusion, https://github.com/pbonazzi/guassian-diffusion, fetched 2026-10-28 (unofficial)
