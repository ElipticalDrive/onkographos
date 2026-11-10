# Near-Far Linear/Disparity Ray Generation with Stratification

Also called: Near-Far Linear Ray Generation with Stratification, Stratified Sampling Along Rays

## Overview
This method generates sample points along camera rays for volume rendering by computing depth values ($z$) either linearly or in disparity space ($1/z$), with optional stratification to reduce sampling variance [S1, unofficial]. It extends the basic linear ray generation by allowing sampling in disparity space, which is often more appropriate for scenes with large depth variations, and by introducing stratified sampling to ensure uniform coverage within each interval [S1, unofficial].

## How it works
**Inputs:**
- `origins`: Ray origins, shape $[B, 3]$ (batch size $B$, 3D coordinates).
- `directions`: Ray directions, shape $[B, 3]$ (normalized vectors).
- `near`: Float, near clip plane distance.
- `far`: Float, far clip plane distance.
- `num_coarse_samples`: Integer, number of samples to generate along each ray.
- `use_linear_disparity`: Boolean, if True, samples linearly in disparity ($1/z$) rather than depth ($z$).
- `use_stratified_sampling`: Boolean, if True, adds uniform random noise to sample positions within intervals to reduce variance.

**Outputs:**
- `points`: 3D sample points, shape $[B, N, 3]$ where $N = \text{num\_coarse\_samples}$.

**Data Structures:**
- `t_vals`: 1D array of shape $[N]$, representing normalized positions in $[0, 1]$.
- `z_vals`: 1D array of shape $[N]$, representing depth or disparity values.
- `mids`: 1D array of shape $[N-1]$, midpoints of intervals.
- `upper`: 1D array of shape $[N]$, upper bounds of intervals.
- `lower`: 1D array of shape $[N]$, lower bounds of intervals.
- `t_rand`: 1D array of shape $[N]$, random uniform noise in $[0, 1]$.

**Steps:**
1. Compute `t_vals` as a linear space from $0.0$ to $1.0$ with `num_coarse_samples` steps.
2. Compute `z_vals` based on `use_linear_disparity`:
   - If `use_linear_disparity` is False: $z_{val} = near \cdot (1 - t) + far \cdot t$.
   - If `use_linear_disparity` is True: $z_{val} = \frac{1}{\frac{1}{near}(1 - t) + \frac{1}{far}t}$.
3. If `use_stratified_sampling` is True:
   - Compute midpoints: $mids = 0.5 \cdot (z_{vals}[1:] + z_{vals}[:-1])$.
   - Compute upper bounds: $upper = [mids, z_{vals}[-1]]$.
   - Compute lower bounds: $lower = [z_{vals}[0], mids]$.
   - Generate random uniform noise: $t_{rand} \sim U[0, 1]$.
   - Compute stratified `z_vals`: $z_{vals} = lower + (upper - lower) \cdot t_{rand}$.
4. If `use_stratified_sampling` is False:
   - Broadcast `z_vals` to shape $[B, N]$.
5. Compute 3D points: $points = origins + z_{vals} \cdot directions$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_stratified_rays(origins, directions, near, far, num_coarse_samples, use_linear_disparity, use_stratified_sampling):
    """
    Generate sample points along camera rays with optional stratification and disparity sampling.
    
    Args:
        origins: np.ndarray, shape [B, 3], ray origins.
        directions: np.ndarray, shape [B, 3], ray directions.
        near: float, near clip plane distance.
        far: float, far clip plane distance.
        num_coarse_samples: int, number of samples per ray.
        use_linear_disparity: bool, if True, sample in disparity space.
        use_stratified_sampling: bool, if True, use stratified sampling.
        
    Returns:
        points: np.ndarray, shape [B, N, 3], 3D sample points.
    """
    B = origins.shape[0]
    N = num_coarse_samples
    
    # Step 1: Compute t_vals
    t_vals = np.linspace(0.0, 1.0, N)
    
    # Step 2: Compute z_vals
    if use_linear_disparity:
        # Disparity space: 1/z is linear
        # z = 1 / ( (1/near)*(1-t) + (1/far)*t )
        inv_near = 1.0 / near
        inv_far = 1.0 / far
        inv_z = inv_near * (1.0 - t_vals) + inv_far * t_vals
        z_vals = 1.0 / inv_z
    else:
        # Linear depth space
        z_vals = near * (1.0 - t_vals) + far * t_vals
        
    # Step 3: Stratified sampling
    if use_stratified_sampling:
        # Compute midpoints
        mids = 0.5 * (z_vals[1:] + z_vals[:-1])
        
        # Compute upper and lower bounds
        # upper = [mids, z_vals[-1]]
        # lower = [z_vals[0], mids]
        upper = np.concatenate([mids, [z_vals[-1]]])
        lower = np.concatenate([[z_vals[0]], mids])
        
        # Generate random noise
        t_rand = np.random.uniform(0.0, 1.0, N)
        
        # Compute stratified z_vals
        z_vals = lower + (upper - lower) * t_rand
        
    # Step 4: Broadcast z_vals to [B, N]
    z_vals_broadcast = np.broadcast_to(z_vals, (B, N))
    
    # Step 5: Compute 3D points
    # points = origins + z_vals * directions
    # origins shape [B, 3], z_vals_broadcast shape [B, N], directions shape [B, 3]
    # We need to multiply z_vals_broadcast by directions appropriately
    # directions: [B, 3] -> [B, 1, 3]
    # z_vals_broadcast: [B, N] -> [B, N, 1]
    # points: [B, N, 3]
    
    directions_expanded = directions[:, np.newaxis, :]  # [B, 1, 3]
    z_vals_expanded = z_vals_broadcast[:, :, np.newaxis]  # [B, N, 1]
    
    points = origins[:, np.newaxis, :] + z_vals_expanded * directions_expanded
    
    return points
```

**Parameters:**
- `near`: Float, near clip plane distance. Typical value: $0.1$ [S1, unofficial].
- `far`: Float, far clip plane distance. Typical value: $100.0$ [S1, unofficial].
- `num_coarse_samples`: Integer, number of samples to generate along each ray. Typical value: $64$ [S1, unofficial].
- `use_linear_disparity`: Boolean, if True, samples linearly in disparity ($1/z$) rather than depth ($z$). Default: False [S1, unofficial].
- `use_stratified_sampling`: Boolean, if True, adds uniform random noise to sample positions within intervals to reduce variance. Default: False [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: When `near` is very small, $1/near$ can become large, leading to numerical instability in disparity calculations [S1, unofficial].
- Boundaries: Ensure `near` < `far` to avoid invalid intervals [S1, unofficial].
- Degenerate Input: If `num_coarse_samples` is 1, stratification may not be meaningful [S1, unofficial].

## Operating characteristics
- Time Complexity: $O(B \cdot N)$, where $B$ is batch size and $N$ is number of samples.
- Memory Complexity: $O(B \cdot N)$ for storing `points`.
- Measured Performance: Not given in the sources.
- Scaling: Linear with respect to batch size and number of samples.
- Strengths: Reduces variance with stratification; disparity sampling handles large depth variations better.
- Limitations: Stratification adds computational overhead; disparity sampling may be less intuitive for some scenes.
- When to Use: When variance reduction is critical or when scenes have large depth variations.
- When Not to Use: When computational resources are limited or scenes have uniform depth distribution.

## Background
This method is part of the ray generation pipeline in volume rendering, specifically designed to improve sampling efficiency and reduce noise [S1, unofficial]. It builds on the concept of stratified sampling, which is a common technique in Monte Carlo integration to reduce variance [S1, unofficial]. The use of disparity space is particularly relevant in computer vision and graphics for handling perspective projections [S1, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
