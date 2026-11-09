# Max Dilation


## Overview
Max Dilation is a resampling technique for step functions that expands the support of non-negative values using max-pooling logic, effectively "blurring" sharp boundaries by considering neighboring intervals. It differs from standard sum-based resampling ([Step Function Resampling](7924-step-function-resampling.md)) by preserving the maximum value within an expanded kernel rather than aggregating mass, which is useful for preventing aliasing or smoothing discrete density fields without introducing negative artifacts.

## How it works
The method operates on a step function defined by boundaries $t$ and values $v$. It expands the influence of each interval by a dilation radius, computes the maximum value within these expanded regions, and maps the result back to the target grid.

**Inputs:**
- $t$: Array of original step boundaries, shape $(N+1,)$.
- $v$: Array of step values, shape $(N,)$. Must be non-negative.
- $t_{target}$: Array of new boundaries for the resampled function, shape $(M+1,)$.
- $dilation$: Scalar radius for dilation.
- $domain$: Tuple $(min, max)$ for clipping boundaries.

**Outputs:**
- $v_{dilated}$: Resampled values on $t_{target}$.

**Data Structures:**
- No complex structures; relies on sorted arrays and vectorized max operations.

**Steps:**
1. **Compute Dilated Boundaries:** For each interval $i$ in the original grid, define the dilated range $[t_i - dilation, t_{i+1} + dilation]$.
   $$ t0_i = t_i - dilation $$
   $$ t1_i = t1_i = t_{i+1} + dilation $$
2. **Sort Combined Endpoints:** Concatenate original boundaries $t$, lower dilated boundaries $t0$, and upper dilated boundaries $t1$. Sort them to create a refined grid $t_{dilate}$.
   $$ t_{combined} = \text{sort}(\text{concat}(t, t0, t1)) $$
   $$ t_{dilate} = \text{clip}(t_{combined}, domain) $$
3. **Compute Dilated Weights/Values:** For each interval in $t_{dilate}$, determine which original intervals overlap with the dilated extent. The value is the maximum of $v$ for all original intervals that have non-zero intersection with the dilated region.
   *Note: The source [S1, unofficial] describes this as "taking max of weights where dilated points fall within original intervals."*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def max_dilate_step_function(t, v, t_target, dilation, domain):
    """
    Resamples a step function using max dilation.
    t: original boundaries (N+1,)
    v: original values (N,)
    t_target: new boundaries (M+1,)
    dilation: float
    domain: (min, max) tuple
    """
    N = len(t) - 1
    M = len(t_target) - 1
    
    # 1. Compute dilated boundaries for each original interval
    # t0[i] is the left edge of interval i dilated left
    # t1[i] is the right edge of interval i dilated right
    t0 = t[:-1] - dilation
    t1 = t[1:] + dilation
    
    # 2. Create a unified set of critical points
    # Combine original, dilated-left, and dilated-right boundaries
    all_points = np.concatenate([t, t0, t1])
    t_dilate = np.sort(all_points)
    
    # 3. Clip to domain
    t_dilate = np.clip(t_dilate, domain[0], domain[1])
    
    # 4. Compute values on the dilated grid
    # For each interval in t_dilate, find the max v from original intervals
    # that overlap with the dilated extent of any original interval covering this point.
    # More precisely: for each point x in t_dilate, find max(v[i]) such that
    # the dilated interval [t[i]-d, t[i+1]+d] contains x.
    
    v_dilated_grid = np.zeros(len(t_dilate) - 1)
    
    for j in range(len(t_dilate) - 1):
        # Interval in dilated grid
        x_start = t_dilate[j]
        x_end = t_dilate[j+1]
        
        max_val = 0.0
        for i in range(N):
            # Dilated extent of original interval i
            d_start = t[i] - dilation
            d_end = t[i+1] + dilation
            
            # Check if dilated extent overlaps with current dilated grid interval
            # Overlap exists if max(x_start, d_start) < min(x_end, d_end)
            if max(x_start, d_start) < min(x_end, d_end):
                if v[i] > max_val:
                    max_val = v[i]
        v_dilated_grid[j] = max_val

    # 5. Map back to t_target
    # Since t_target is a subset or different grid, we need to aggregate.
    # However, the source [S1, unofficial] implies the output is directly on t_target.
    # If t_target != t_dilate, we must resample v_dilated_grid onto t_target.
    # Using the same max-dilation logic recursively or simply sampling.
    # Given the complexity, a common implementation is to treat t_target as the
    # final query grid and apply the max-overlap logic directly against original t.
    
    # Simplified direct mapping to t_target:
    v_out = np.zeros(M)
    for j in range(M):
        x_start = t_target[j]
        x_end = t_target[j+1]
        max_val = 0.0
        for i in range(N):
            d_start = t[i] - dilation
            d_end = t[i+1] + dilation
            if max(x_start, d_start) < min(x_end, d_end):
                if v[i] > max_val:
                    max_val = v[i]
        v_out[j] = max_val
        
    return v_out
```

**Parameters:**
- `dilation`: Typically small relative to grid spacing. Not given in sources.

**Edge Cases:**
- If $v$ contains negative values, the "max" logic may preserve negatives, violating the non-negative assumption often required for density.
- If $dilation$ is very large, the entire domain may collapse to the global maximum.

## Subtopics

### Max Dilation (Morton)
Performs max pooling with Morton coordinates (dilation) on the density grid [S2, unofficial]. It ensures the grid is a contiguous CUDA tensor, calculates $H = \sqrt[3]{H^3}$ (where $H$ is the grid dimension), and calls a backend CUDA kernel `_backend.morton3D_dilation(grid, C, H, grid_dilation)` [S2, unofficial]. The input `grid` is a float tensor of shape $[C, H \times H \times H]$ with $H \% 2 == 0$ [S2, unofficial]. The output `grid_dilation` has the same shape [S2, unofficial]. This variant is optimized for 3D volumetric data using spatial locality via Morton ordering.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$ for the naive implementation where $N$ is original intervals and $M$ is target intervals. The Morton variant uses CUDA kernels for parallelization, likely $O(1)$ per voxel with high constant overhead for setup.
- **Memory Complexity:** $O(N + M)$ for the naive implementation. The Morton variant requires contiguous memory for the CUDA kernel.
- **Strengths:** Preserves peak values; prevents "bleeding" of low values into high-value regions; suitable for non-negative density fields.
- **Limitations:** Can introduce blockiness if dilation is coarse; computationally expensive if $N$ and $M$ are large without spatial indexing.
- **When to use:** When smoothing discrete density fields without losing high-density peaks, or in Gaussian Splatting contexts for initialization [S1, unofficial].

## Background
Max Dilation is used in sparse-view Gaussian Splatting pipelines, specifically in InstantSplat, to handle geometry initialization and density field resampling [S1, unofficial]. The Morton variant is used in ER-NeRF for efficient 3D grid processing [S2, unofficial].

## Sources
[S1] InstantSplat: Sparse-view Gaussian Splatting in Seconds, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
