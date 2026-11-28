# CDF-based Ray Resampling

Also called: CDF Resampling Kernel, CDF Resampling for Ray Points, midpoint resampling

## Overview
CDF-based Ray Resampling redistributes a set of input ray samples into a new, fixed-number set of samples by sampling from the Cumulative Distribution Function (CDF) of the volumetric rendering weights [S1, unofficial]. It differs from standard implementations by sampling the midpoints of the resampled intervals rather than the start or end points, and by explicitly adding a background interval to the CDF to allow samples to be assigned to the background when foreground density is low [S1, unofficial]. This method builds on [Refined Ray Generation via PDF Sampling](6837-refined-ray-generation-via-pdf-sampling.md) by providing a deterministic, weight-aware resampling strategy that preserves the total energy of the ray while adapting sample density to the signal [S1, unofficial].

## How it works
The method takes a set of rays with varying numbers of input samples and resamples each ray to a target number of output samples (`steps`). It operates on a per-ray basis using a CUDA kernel, but the logical steps are defined below.

### Inputs and Outputs
**Inputs:**
*   `packed_info`: Tensor of shape `[n_rays, 2]`. Column 0 is the base index in the flat arrays for each ray. Column 1 is the number of input samples (`num_steps`) for each ray.
*   `t_starts`: Tensor of shape `[n_samples_in, 1]`. The start positions ($t_{start}$) of the input frustum-shaped samples along each ray.
*   `t_ends`: Tensor of shape `[n_samples_in, 1]`. The end positions ($t_{end}$) of the input frustum-shaped samples along each ray.
*   `weights`: Tensor of shape `[n_samples_in]`. The volumetric rendering weights ($w_i$) for each input sample.
*   `steps`: Integer. The target number of resampled points per ray.

**Outputs:**
*   `resample_ts`: Tensor of shape `[total_resampled_samples, 1]`. The $t$-coordinates (midpoints) of the resampled samples.
*   `resample_offsets`: Tensor of shape `[total_resampled_samples, 1]`. The offset of each resampled midpoint relative to the start of its assigned input interval.
*   `resample_indices`: Tensor of shape `[total_resampled_samples]` (int64). The index of the input interval (from `weights`) that each resampled sample falls into.
*   `resample_fg_counts`: Tensor of shape `[n_samples_in]` (int32). The number of resampled samples assigned to each input foreground interval.
*   `resample_bg_counts`: Tensor of shape `[n_rays]` (int32). The number of resampled samples assigned to the background for each ray.
*   `resample_packed_info`: Tensor of shape `[n_rays, 2]`. Column 0 is the start index in the output arrays. Column 1 is the end index (exclusive) in the output arrays.

### Data Structures
The algorithm uses flat tensors for efficiency. The `packed_info` structure allows random access to ray segments without iterating through a jagged array. The output arrays are pre-allocated based on the total number of resampled steps to avoid dynamic resizing.

### Algorithm Steps
1.  **Validation**: Assert that `n_samples` (the number of input samples per ray, assumed uniform or handled via `packed_info`) is greater than 1. If `n_samples == 1`, the method is undefined [S1, unofficial].
2.  **Pre-computation**:
    *   Calculate `total_samples`: Sum of `num_steps` for all rays (from `packed_info`).
    *   Calculate `resample_num_steps`: For each ray, the number of steps to resample. In the reference implementation, this is often uniform (`steps`), but the logic supports variable counts. Let $N_{out}$ be the target steps per ray.
    *   Calculate `resample_cum_steps`: Cumulative sum of `resample_num_steps` across rays. This defines the boundaries in the output tensors.
    *   Allocate output tensors:
        *   `total_steps` = Sum of `resample_num_steps`.
        *   `resample_ts`, `resample_offsets` of size `total_steps`.
        *   `resample_indices` of size `total_steps`.
        *   `resample_fg_counts` of size `total_samples`.
        *   `resample_bg_counts` of size `n_rays`.
3.  **Per-Ray Processing (Kernel)**: For each ray $i$:
    *   **Locate Segment**: Get base index $B_i = \text{packed\_info}[i, 0]$ and step count $S_i = \text{packed\_info}[i, 1]$.
    *   **Compute Total Weight**:
        *   Sum foreground weights: $W_{fg} = \sum_{j=0}^{S_i-1} w_{B_i+j}$.
        *   Compute background weight: $W_{bg} = \max(1.0 - W_{fg}, 0.0)$.
        *   Total weight: $W_{total} = W_{fg} + W_{bg}$.
    *   **Calculate CDF Step Size**:
        *   Let $N_{out}$ be the number of resampled steps for this ray.
        *   $cdf\_step\_size = \frac{1.0 - \frac{1.0}{N_{out}}}{N_{out} - 1}$.
        *   This formula ensures that the sampling points are uniformly distributed in CDF space, excluding the very end of the CDF to handle the background interval correctly [S1, unofficial].
    *   **Initialize Pointers**:
        *   $cdf\_prev = 0.0$.
        *   $cdf\_next = \frac{w_{B_i}}{W_{total}}$.
        *   $cdf\_u = \frac{1.0}{2 \cdot N_{out}}$. This is the first sampling position in CDF space (midpoint of the first bin).
    *   **Resample Loop**: For $k = 0$ to $N_{out} - 1$:
        *   **Find Interval**: While $cdf\_u > cdf\_next$ and $cdf\_next < 1.0$:
            *   $cdf\_prev = cdf\_next$.
            *   If $cdf\_prev + cdf\_step\_size < 1.0$:
                *   $cdf\_next = cdf\_prev + cdf\_step\_size$.
            *   Else:
                *   $cdf\_next = 1.0$.
            *   *Note*: The reference implementation uses a specific logic where `cdf_next` is updated based on the weight sum. A more robust interpretation based on the notes:
                *   Iterate through input weights $w_j$ (where $j$ is the index in the ray's segment).
                *   Accumulate weight sum.
                *   If $cdf\_u$ falls within the cumulative weight range of an input interval, assign the sample to that interval.
        *   **Assign Sample**:
            *   If $cdf\_u$ corresponds to the background interval (i.e., it exceeds the sum of all foreground weights):
                *   Assign to background.
                *   $t_{mid} = t_{end, last} + 10000.0$ (fixed offset for background, as per [S1, unofficial]).
                *   Increment `resample_bg_counts[i]`.
                *   Set `resample_indices` to a special value or handle separately.
            *   Else:
                *   Identify the input interval $m$ such that $CDF(m-1) \le cdf\_u \le CDF(m)$.
                *   Calculate the local position within the interval: $local\_u = \frac{cdf\_u - CDF(m-1)}{w_m}$.
                *   Calculate midpoint: $t_{mid} = \frac{t_{start, m} + t_{end, m}}{2}$. (Note: The notes specify sampling midpoints. If the CDF sampling lands in an interval, the output $t$ is the midpoint of that input interval, or potentially interpolated. The notes say "sample midpoints instead of start/end points", implying the output $t$ is the midpoint of the *resampled* interval? No, it says "sample midpoints... of the resampled frustum-shape samples". Given the input is frustum-shaped, the output is likely the midpoint of the input interval that the CDF sample falls into, or a linear interpolation. The notes explicitly state: "Calculate the number of steps per ray... Compute cumulative sum... Construct resample_packed_info... Allocate output tensors... Launch kernel". The kernel calculates `resample_ts`. The pitfall is `n_samples > 1`. The key distinction is "sampling midpoints". This usually means if a sample falls in interval $[t_i, t_{i+1}]$, its $t$ value is $(t_i + t_{i+1})/2$.
                *   Calculate offset: $offset = t_{mid} - t_{start, m}$.
                *   Increment `resample_fg_counts[m]`.
                *   Store $t_{mid}$ in `resample_ts`, $offset$ in `resample_offsets`, and $m$ in `resample_indices`.
        *   **Advance CDF**: $cdf\_u += cdf\_step\_size$.
4.  **Finalize**:
    *   Fill remaining points in the last interval (background) if the loop ends with $j < num\_bins$ (this part of the note is slightly ambiguous, but implies handling the tail of the CDF).
    *   Return the populated tensors.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def cdf_resampling_kernel(packed_info, t_starts, t_ends, weights, steps):
    """
    Simulates the CDF-based Ray Resampling kernel logic.
    Note: This is a Python simulation for clarity. The actual implementation is in CUDA.
    """
    n_rays = packed_info.shape[0]
    resample_ts = []
    resample_offsets = []
    resample_indices = []
    resample_fg_counts = torch.zeros(weights.shape[0], dtype=torch.int32)
    resample_bg_counts = torch.zeros(n_rays, dtype=torch.int32)
    resample_packed_info = []
    
    current_out_idx = 0
    
    for i in range(n_rays):
        base_idx = packed_info[i, 0].item()
        num_steps = packed_info[i, 1].item()
        
        # Extract ray-specific data
        ray_weights = weights[base_idx : base_idx + num_steps]
        ray_t_starts = t_starts[base_idx : base_idx + num_steps]
        ray_t_ends = t_ends[base_idx : base_idx + num_steps]
        
        # Compute total weight including background
        w_fg_sum = ray_weights.sum().item()
        w_bg = max(1.0 - w_fg_sum, 0.0)
        w_total = w_fg_sum + w_bg
        
        if w_total == 0:
            # Degenerate case: no density. Assign all to background?
            # Or uniform? The notes imply background handling when foreground is low.
            # If w_total is 0, CDF is undefined. Assume uniform or skip.
            # For robustness, let's assume uniform distribution over background.
            pass 
            
        # CDF Step Size
        # cdf_step_size = (1.0 - 1.0 / num_bins) / (num_bins - 1)
        # Here num_bins = steps
        if steps <= 1:
            raise ValueError("steps must be > 1")
            
        cdf_step_size = (1.0 - 1.0 / steps) / (steps - 1)
        
        # Initialize CDF pointers
        cdf_prev = 0.0
        # cdf_next is the cumulative weight up to the current interval
        # We iterate through intervals
        cdf_u = 1.0 / (2 * steps) # First sample midpoint in CDF space
        
        ray_resampled_ts = []
        ray_resampled_offsets = []
        ray_resampled_indices = []
        
        # Build CDF of foreground weights
        cdf_fg = torch.cumsum(ray_weights, dim=0)
        
        for k in range(steps):
            # Determine which interval cdf_u falls into
            # Normalize cdf_u to [0, w_total]
            # Actually, cdf_u is in [0, 1] relative to w_total?
            # The notes say: cdf_next = weights[0] / weights_sum.
            # So cdf_u is in [0, 1].
            
            # Find index m such that cdf_fg[m-1]/w_total < cdf_u <= cdf_fg[m]/w_total
            # Handle background
            if cdf_u > (w_fg_sum / w_total) if w_total > 0 else 0:
                # Background sample
                # Fixed offset 10000.f added to ends[steps-1]
                last_t_end = ray_t_ends[-1].item()
                t_mid = last_t_end + 10000.0
                offset = 10000.0 # Offset from last end? Or relative to start?
                # Notes say "offsets of the resampled midpoints with respect to the starts of the sampled intervals"
                # If it's background, there is no "interval" in the foreground sense.
                # Let's assume offset is 0 or handled separately.
                # The notes say "fill remaining points in the last interval (background) with fixed offset 10000.f added to ends[steps-1]"
                # This implies the t value is ends[-1] + 10000.
                # Offset might be 10000.
                
                ray_resampled_ts.append(t_mid)
                ray_resampled_offsets.append(10000.0)
                ray_resampled_indices.append(-1) # Special index for background
                resample_bg_counts[i] += 1
            else:
                # Foreground sample
                # Find interval
                # cdf_u * w_total
                target_weight = cdf_u * w_total
                
                # Find first index where cdf_fg >= target_weight
                # Use searchsorted
                m = torch.searchsorted(cdf_fg, target_weight, side='right').item()
                # Clamp m
                m = min(m, num_steps - 1)
                
                # Get interval bounds
                t_start = ray_t_starts[m].item()
                t_end = ray_t_ends[m].item()
                
                # Sample midpoint
                t_mid = (t_start + t_end) / 2.0
                
                # Offset from start
                offset = t_mid - t_start
                
                ray_resampled_ts.append(t_mid)
                ray_resampled_offsets.append(offset)
                ray_resampled_indices.append(base_idx + m)
                resample_fg_counts[base_idx + m] += 1
            
            # Advance CDF
            cdf_u += cdf_step_size
            
        # Store results
        resample_ts.extend(ray_resampled_ts)
        resample_offsets.extend(ray_resampled_offsets)
        resample_indices.extend(ray_resampled_indices)
        
        # Packed info for output
        resample_packed_info.append([current_out_idx, current_out_idx + steps])
        current_out_idx += steps
        
    return (
        torch.tensor(resample_ts, dtype=torch.float32).unsqueeze(1),
        torch.tensor(resample_offsets, dtype=torch.float32).unsqueeze(1),
        torch.tensor(resample_indices, dtype=torch.int64),
        resample_fg_counts,
        resample_bg_counts,
        torch.tensor(resample_packed_info, dtype=torch.int64)
    )
```

### Parameters
*   `steps`: Target number of resampled points per ray. Must be greater than 1.
*   `packed_info`: Tensor of shape `[n_rays, 2]`.
*   `weights`: Tensor of shape `[n_samples_in]`.
*   `t_starts`, `t_ends`: Tensors of shape `[n_samples_in, 1]`.

### Edge Cases and Pitfalls
*   **n_samples == 1**: The CUDA kernel does not handle rays with only 1 input sample. The Python wrapper asserts `n_samples > 1` [S1, unofficial].
*   **Zero Weight**: If all weights are zero, $W_{total} = 0$. The CDF is undefined. The implementation should handle this by assigning all samples to the background or distributing them uniformly. The notes mention `max(1.0 - weights_sum, 0.0)`, which handles the case where foreground weight is low but not zero. If foreground is exactly zero, background is 1.0.
*   **Background Interval**: The background interval is handled by adding a fixed offset of 10000.0 to the last end position. This is a heuristic to push background samples far away from the foreground [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N_{rays} \cdot N_{out})$ for the resampling step, where $N_{out}$ is the number of resampled steps per ray. The CDF construction is $O(N_{in})$ per ray.
*   **Memory Complexity**: $O(N_{in} + N_{out})$ for storing input and output tensors.
*   **Strengths**: Deterministic resampling based on weight distribution. Handles background explicitly. Efficient CUDA implementation.
*   **Limitations**: Requires `n_samples > 1`. The fixed background offset is a heuristic.
*   **When to use**: When you need to adapt the number of samples per ray based on the signal density, while maintaining a fixed output size for downstream processing.

## Background
This method is implemented in the RISE-SDF repository [S1, unofficial]. It is a specialized ray resampling technique for volume rendering, specifically designed to work with frustum-shaped samples and to handle background contributions explicitly.

## Sources
[S1] dehezhang2/RISE-SDF, https://github.com/dehezhang2/RISE-SDF, fetched 2026-10-27 (unofficial)
