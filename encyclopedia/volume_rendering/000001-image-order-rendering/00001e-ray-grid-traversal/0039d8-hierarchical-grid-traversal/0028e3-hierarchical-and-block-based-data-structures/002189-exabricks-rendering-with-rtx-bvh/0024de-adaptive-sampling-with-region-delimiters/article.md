# Adaptive Sampling with Region Delimiters

Also called: Region-based Adaptive Sampling

## Overview
Adaptive Sampling with Region Delimiters is a sampling strategy for Adaptive Block-Refined (ABR) volume data that adapts the sampling rate to the finest cell size within each region, using sample positions as delimiters to handle boundary artifacts [S1]. Unlike Ljung [23], which defines a global set of finest level samples skipped by coarseness, this method defines independent sampling intervals for each region [S1]. It solves the problem of boundary artifacts caused by successive regions sampled at different rates, where the distance between the last sample of one region and the first of the next can be smaller or larger than either region's sampling rate [S1].

## How it works
**Inputs and Outputs:**
- Input: ABR region metadata (specifically the smallest cell size within the region), ray parameters (origin, direction, entry/exit points $t_{in}, t_{out}$), and a user-provided scaling parameter [S1].
- Output: A sequence of sample positions and values within the region, integrated into the ray's color/opacity accumulation [S1].

**Data Structures:**
- Region Metadata: Contains the minimum cell size $\rho_{min}$ within the ABR block [S1].
- Ray State: Tracks the current position $t$ and a per-ray random offset $\rho$ [S1].

**Parameters:**
- Base sampling rate: 2 samples per smallest cell size in the region [S1].
- User-provided scaling parameter: Scales the base sampling rate to trade quality for performance [S1].
- $\rho$: Per-ray random offset for interleaved sampling [S1].

**Steps:**
1. **Determine Sampling Step ($dt$):** Calculate the base sampling step $dt_{base}$ as $\rho_{min} / 2$ to achieve two samples per smallest cell size [S1]. Apply the user scaling factor $S$ to get the final step size $dt = S \times dt_{base}$ [S1].
2. **Calculate First Sample Position ($t_0$):** Compute the first sample distance using the formula $t_0 = dt \cdot \lceil \rho / dt \rceil$ [S1].
3. **Generate Sample Positions:** Step the ray through the region, sampling at each subsequent position $t_{i+1} = t_i + dt$ [S1].
4. **Define Delimiters and Sub-intervals:** Treat the generated sample positions $t_0, t_1, \dots, t_n$ as delimiters [S1]. These delimiters split the region's overlap interval $[t_{in}, t_{out}]$ into sub-intervals $T_0=[t_{in}, t_0], T_1=[t_0, t_1], \dots, T_m=[t_n, t_{out}]$ [S1].
5. **Sample Sub-intervals:** For each sub-interval $T_i$, sample at its midpoint $(T_i[0] + T_i[1]) / 2$ [S1]. Weigh the sample in the opacity correction term by the interval length $dt_i = |T_i|$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_region_with_delimiters(ray_origin, ray_direction, t_in, t_out, region_min_cell_size, scaling_factor, ray_random_offset):
    """
    Samples a volume region using adaptive sampling with region delimiters.
    
    Args:
        ray_origin: vec3
        ray_direction: vec3 (normalized)
        t_in: float (entry distance)
        t_out: float (exit distance)
        region_min_cell_size: float (smallest cell size in the ABR region)
        scaling_factor: float (user-provided scaling parameter)
        ray_random_offset: float (per-ray random offset rho)
        
    Returns:
        list of (position, opacity_contribution, interval_length) tuples
    """
    # Step 1: Determine Sampling Step
    # Base rate is 2 samples per smallest cell size, so step = size / 2 [S1]
    dt_base = region_min_cell_size / 2.0
    dt = scaling_factor * dt_base
    
    # Step 2: Calculate First Sample Position
    t0 = dt * math.ceil(ray_random_offset / dt)
    t0 = max(t_in, t0)  # Ensure first delimiter is within region bounds [G]
    
    # Step 3: Generate Sample Positions (Delimiters)
    delimiters = []
    t_current = t0
    while t_current < t_out:
        delimiters.append(t_current)
        t_current += dt
    
    # Step 4: Define Sub-intervals
    all_points = [t_in] + delimiters + [t_out]
    
    samples = []
    
    # Step 5: Sample Sub-intervals
    for i in range(len(all_points) - 1):
        t_start = all_points[i]
        t_end = all_points[i+1]
        
        if t_end <= t_start:
            continue
            
        # Midpoint sampling [S1]
        midpoint = (t_start + t_end) / 2.0
        position = ray_origin + ray_direction * midpoint
        
        # Interval length for weighting [S1]
        dt_i = t_end - t_start
        
        # [G] Standard volume sampling at the computed position
        value = sample_volume(position)
        
        # [G] Standard transfer function mapping value to opacity
        alpha = transfer_function(value)
        
        # Opacity Correction: alpha_tilde = 1 - (1 - alpha)^(dt_i / dt) [S1]
        if alpha > 0:
            alpha_corrected = 1.0 - (1.0 - alpha) ** (dt_i / dt)
        else:
            alpha_corrected = 0.0
            
        samples.append((position, alpha_corrected, dt_i))
        
    return samples

# Helper definitions [G]
def sample_volume(position):
    # Returns density/value at position from the volume grid
    return 0.0

def transfer_function(value):
    # Returns opacity alpha for the given density/value
    return 0.0
```

**Edge Cases and Pitfalls:**
- **Boundary Artifacts:** Successive regions sampled at different rates can have distances between the last sample of one and first of the next that are smaller or larger than the region's sampling rate, causing artifacts [S1]. The delimiter method mitigates this by explicitly sampling the gap intervals [S1].
- **Small Overlap Intervals:** The method takes at least one sample for each region hit, even if the overlap interval is small [S1].
- **Banding Artifacts:** At lower sampling rates, biasing the sample positions (using midpoints of variable intervals) can result in slight banding artifacts [S1].

## Subtopics

### Opacity Correction for Variable Sampling Rate
A correction factor applied to opacity to account for variable step sizes between samples, preventing incorrect opacity due to undersampling [S1]. The formula is $\tilde{\alpha} = 1 - (1 - \alpha)^{s/s_1}$, where $s$ is the current step size (interval length $dt_i$), $s_1$ is the base step size ($dt$), and $\alpha$ is the opacity from the transfer function [S1]. Compared to standard adaptive sampling, this correction takes one additional sample per region (for the boundary intervals) and prevents opacity decrease from undersampling, at the cost of slight banding artifacts at low sampling rates [S1].

## Operating characteristics
- **Time Complexity:** Proportional to the number of sub-intervals created by the delimiters [S1]. Since each region generates samples based on its finest cell size, fine regions require more samples [S1]. The delimiter approach adds overhead by requiring sampling at the boundaries of each region (at least one sample per region hit) [S1].
- **Memory Complexity:** Low, as it is a streaming method [S1]. No additional large data structures are required beyond the region metadata and ray state [S1].
- **Measured Performance:** The method relies on the underlying BVH traversal performance [S1]. BVH rebuilds for large datasets (e.g., Exajet) take roughly 300ms on a Titan RTX or RTX 8000 GPU [S1].
- **Strengths:** Eliminates boundary artifacts between regions of different refinement levels; adapts sampling to local data frequency without guessing [S1].
- **Limitations:** Can introduce banding artifacts at low sampling rates; requires at least one sample per region hit, which may be inefficient for very small overlaps [S1].
- **When to Use:** When rendering ABR data with significant variation in cell sizes across adjacent regions, and visual continuity at region boundaries is critical [S1].
- **When Not to Use:** When performance is paramount and slight boundary artifacts are acceptable, or when regions are very small and the overhead of boundary sampling dominates [S1].

## Background
This method was introduced in the context of ray tracing structured AMR (Adaptive Mesh Refinement) data using ExaBricks [S1]. It addresses limitations in previous multi-block adaptive sampling techniques, such as those by Ljung [23], by providing independent sampling intervals per region [S1]. The approach leverages ABR metadata to adapt sampling rates to the finest cell size, avoiding the need to guess data frequency within a region [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
