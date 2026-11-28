# Interval Sampling

Also called: Sample Intervals

## Overview
Interval Sampling is a step function resampling method that derives interval boundaries from sampled centers rather than evaluating the function at discrete points [S1, S2, unofficial]. It ensures resulting intervals are symmetric around the chosen centers by reflecting adjacent midpoints to determine boundaries [S1, S2, unofficial]. It operates within a specified domain and is distinct from sibling methods like Max Dilation and CDF Integration [S1, S2, unofficial].

## How it works
**Inputs:**
- `centers`: A tensor of shape `(..., N)` containing the sampled centers $c_i$ for $N$ samples [S1, S2, unofficial].
- `domain`: A tuple `(minval, maxval)` specifying the valid range for the output boundaries [S2, unofficial].
- `num_samples`: The number of samples $N$ [S1, S2, unofficial].

**Outputs:**
- `t_samples`: A tensor of shape `(..., N+1)` containing the interval boundaries $t_j$ [S1, S2, unofficial].

**Data Structures:**
- `centers`: Floating-point tensor representing the centers of the intervals [S1, S2, unofficial].
- `mid`: Floating-point tensor representing the midpoints between adjacent centers [S1, S2, unofficial].
- `first`, `last`: Floating-point tensors representing the clamped first and last boundaries [S1, S2, unofficial].
- `t_samples`: Concatenated tensor of boundaries [S1, S2, unofficial].

**Steps:**
1. **Validation**: Check if `num_samples > 1`; raise a `ValueError` otherwise [S2, unofficial].
2. **Center Sampling**: Obtain `centers` using a deterministic sampler: `centers = sample(..., deterministic_center=True)` [S1, S2, unofficial].
3. **Midpoint Calculation**: Compute the midpoints between adjacent centers:
   $$ mid = \frac{centers[..., 1:] + centers[..., :-1]}{2} $$
4. **First Boundary**: Reflect the first midpoint around the first center and clamp to `minval`:
   $$ first = \max(minval, 2 \cdot centers[..., :1] - mid[..., :1]) $$
5. **Last Boundary**: Reflect the last midpoint around the last center and clamp to `maxval`:
   $$ last = \min(maxval, 2 \cdot centers[..., -1:] - mid[..., -1:]) $$
6. **Concatenation**: Concatenate `first`, `mid`, and `last` along the last dimension to form the final boundaries:
   $$ t\_samples = \text{concatenate}([first, mid, last], \text{dim}=-1) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interval_sampling(centers, domain):
    """
    Resamples a step function by deriving interval boundaries from centers.
    
    Args:
        centers: Tensor of shape (..., N) containing sample centers.
        domain: Tuple (minval, maxval) for clamping boundaries.
        
    Returns:
        t_samples: Tensor of shape (..., N+1) containing interval boundaries.
    """
    # Step 1: Validation
    if centers.shape[-1] <= 1:
        raise ValueError("num_samples must be greater than 1")
    
    minval, maxval = domain
    
    # Step 2: Center Sampling (Assumed to be provided as input)
    # centers = sample(..., deterministic_center=True)
    
    # Step 3: Midpoint Calculation
    mid = (centers[..., 1:] + centers[..., :-1]) / 2.0
    
    # Step 4: First Boundary
    first = (2.0 * centers[..., :1] - mid[..., :1]).clamp(min=minval)
    
    # Step 5: Last Boundary
    last = (2.0 * centers[..., -1:] - mid[..., -1:]).clamp(max=maxval)
    
    # Step 6: Concatenation
    t_samples = concatenate([first, mid, last], dim=-1)
    
    return t_samples

def concatenate(tensors, dim):
    """Concatenate tensors along the specified dimension [G]."""
    import numpy as np
    return np.concatenate(tensors, axis=dim)
```

**Parameters:**
- `domain`: Tuple `(minval, maxval)` specifying the range of valid values for $t$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- If `num_samples <= 1`, the method raises a `ValueError` [S1, S2, unofficial].
- Clamping ensures boundaries stay within the valid domain, preventing out-of-bounds errors [S1, S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Not given in the sources.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Not given in the sources.

**Strengths and Limitations:**
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.

**When to Use:**
- Not given in the sources.

**When Not to Use:**
- Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] rongduo/DOFGS_implementation, https://github.com/rongduo/DOFGS_implementation, fetched 2026-10-27 (unofficial)
