# Log-space Exclusive Cumulative Sum for Transmittance

Also called: Log-space Exclusive Cumulative Sum, Log-space Exclusive Scan for Transmittance

## Overview
This method computes the exclusive cumulative sum of transmittance values in logarithmic space to avoid underflow and enable efficient scatter-add operations for alpha compositing weights [S1, unofficial]. It differs from standard linear-space exclusive scans by operating on $\log(1-\alpha)$, which preserves precision for small alpha values, and by explicitly handling ray-segment boundaries to reset cumulative state between independent rays [S1, unofficial]. It builds on [Exclusive Prefix Sum (Scan)](3007-exclusive-prefix-sum-scan.md) [G].

## How it works
**Inputs:**
- `alphas`: Array of alpha values for each intersection, shape $[M]$.
- `ray_indices`: Array of ray IDs corresponding to each intersection, shape $[M]$, integer type.

**Outputs:**
- `transmittances`: Array of transmittance values $T_i$, shape $[M]$.
- `weights`: Array of compositing weights $w_i = T_i \cdot \alpha_i$, shape $[M]$.

**Data Structures:**
- `log_oma`: Float array $[M]$, storing $\log(1-\alpha_i)$ [S1, unofficial].
- `log_cumsum`: Float array $[M]$, storing inclusive cumulative sum of `log_oma` [S1, unofficial].
- `log_exclusive`: Float array $[M]$, storing exclusive cumulative sum of `log_oma` [S1, unofficial].
- `is_start`: Boolean array $[M]$, marking the first intersection of each ray [S1, unofficial].
- `segment_ids`: Integer array $[M]$, identifying the segment (ray) index for each element [S1, unofficial].
- `corrections_compact`: Float array $[K]$, where $K$ is the number of segments, storing the exclusive cumulative sum value at the start of each segment [S1, unofficial].

**Steps:**

1.  **Log-Transform:** Compute the log of one-minus-alpha for each intersection to stabilize values near zero [S1, unofficial].
    $$ \text{log\_oma}_i = \log(1 - \alpha_i) $$
    *Pitfall:* If $\alpha_i = 1$, $\log(0)$ results in $-\infty$. Clamp $\alpha_i$ to a maximum value slightly less than 1 (e.g., $1 - \epsilon$) before logging [G].

2.  **Inclusive Scan:** Compute the global inclusive cumulative sum of `log_oma` [S1, unofficial].
    $$ \text{log\_cumsum}_i = \sum_{j=0}^{i} \text{log\_oma}_j $$

3.  **Exclusive Shift:** Convert the inclusive sum to an exclusive sum by shifting the array right and prepending zero [S1, unofficial].
    $$ \text{log\_exclusive}_i = \begin{cases} 0 & i=0 \\ \text{log\_cumsum}_{i-1} & i>0 \end{cases} $$

4.  **Boundary Detection:** Identify the start of each new ray by comparing the current ray index with the previous ray index [S1, unofficial].
    - Create `shifted_rays` by concatenating a dummy value (`ray_indices[0] - 1`) to the front of `ray_indices[:-1]` [S1, unofficial].
    - Compute `is_start`:
      $$ \text{is\_start}_i = (\text{ray\_indices}_i \neq \text{shifted\_rays}_i) $$

5.  **Segment ID Assignment:** Compute a unique ID for each ray segment using the cumulative sum of the start mask [S1, unofficial].
    $$ \text{segment\_ids}_i = \left(\sum_{j=0}^{i} \text{is\_start}_j\right) - 1 $$
    *Note:* This assumes `is_start` is cast to integer (0 or 1) before summation [G].

6.  **Correction Value Extraction:** Extract the exclusive cumulative sum value at the start of each segment [S1, unofficial]. These values represent the offset that must be subtracted to reset the cumulative sum to zero at each ray boundary [S1, unofficial].
    - Let `corrections_compact` be the values of `log_exclusive` at indices where `is_start` is true [S1, unofficial].
    - Gather the correction for each element:
      $$ \text{correction}_i = \text{corrections\_compact}_{\text{segment\_ids}_i} $$

7.  **Boundary Reset:** Subtract the correction from the exclusive cumulative sum to ensure transmittance starts at 1.0 (log 0) for each new ray [S1, unofficial].
    $$ \text{log\_transmittance}_i = \text{log\_exclusive}_i - \text{correction}_i $$

8.  **Exponentiation:** Convert back to linear space [S1, unofficial].
    $$ \text{transmittances}_i = \exp(\text{log\_transmittance}_i) $$

9.  **Weight Calculation:** Compute the final compositing weights [S1, unofficial].
    $$ \text{weights}_i = \text{transmittances}_i \cdot \alpha_i $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def log_space_exclusive_cumsum(alphas, ray_indices):
    """
    Compute transmittance and weights using log-space exclusive cumulative sum.
    
    Args:
        alphas: np.array of shape [M], float32, values in [0, 1).
        ray_indices: np.array of shape [M], int32, ray IDs.
        
    Returns:
        transmittances: np.array of shape [M], float32.
        weights: np.array of shape [M], float32.
    """
    # Step 1: Log-transform
    # [G] Clamp alphas to avoid log(0) if necessary.
    log_oma = np.log(1.0 - alphas)
    
    # Step 2: Inclusive cumulative sum
    log_cumsum = np.cumsum(log_oma)
    
    # Step 3: Convert to exclusive by shifting
    # Prepend 0.0, drop last element
    log_exclusive = np.concatenate([np.zeros(1), log_cumsum[:-1]])
    
    # Step 4: Identify segment boundaries
    # Shift ray_indices right by 1, filling first element with a value != ray_indices[0]
    shifted_rays = np.concatenate([ray_indices[:1] - 1, ray_indices[:-1]])
    is_start = ray_indices != shifted_rays
    
    # Step 5: Compute per-segment correction IDs
    # Cast bool to int32 for cumsum
    segment_ids = np.cumsum(is_start.astype(np.int32)) - 1
    
    # Step 6: Gather correction values
    # corrections_compact contains log_exclusive values at the start of each segment
    start_mask_indices = np.where(is_start)[0]
    corrections_compact = log_exclusive[start_mask_indices]
    
    # Gather correction for each element
    corrections = corrections_compact[segment_ids]
    
    # Step 7: Reset boundary values
    log_transmittance = log_exclusive - corrections
    
    # Step 8: Exponentiate
    transmittances = np.exp(log_transmittance)
    
    # Step 9: Compute weights
    weights = transmittances * alphas
    
    return transmittances, weights
```

**Parameters:**
- No explicit tuning parameters. Precision depends on floating-point type (typically `float32` or `float64`) [G].

**Edge Cases and Pitfalls:**
- **Alpha = 1.0:** $\log(1-1) = -\infty$. This breaks the cumulative sum. Clamp $\alpha$ to $1-\epsilon$ [G].

## Operating characteristics
Not given in the sources.

## Background
This method is used in differentiable Gaussian splatting rasterization, specifically in implementations like `gsplat-mlx`, to compute transmittance and weights for alpha compositing [S1, unofficial]. It addresses the numerical instability of standard alpha compositing when dealing with many small alpha values, which can lead to underflow in linear space [S1, unofficial]. By working in log space, the method ensures that transmittance values remain representable and accurate [G].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
