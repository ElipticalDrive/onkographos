# Entry Align

Also called: Half-step Entry Align

## Overview
Entry Align is a sample placement strategy for fixed-step volume raycasting that shifts the first sample point half a step away from the volume entry boundary to align with the Riemann sum midpoint estimate [S1, unofficial]. Unlike its sibling [Eye Align](849-eye-align.md), which aligns samples relative to the eye position, Entry Align aligns the sampling grid relative to the volume geometry, eliminating the boundary estimation error at the entry point [S1, unofficial].

## How it works
**Inputs:**
- `entryPoint`: The 3D position where the ray enters the volume, in texture coordinates.
- `exitPoint`: The 3D position where the ray exits the volume, in texture coordinates.
- `dim`: The dimensions of the volume texture (width, height, depth).
- `samplingRate`: The number of samples per unit length in texture space.

**Outputs:**
- `tStart`: The starting parameter for the first sample along the ray.
- `tIncr`: The step increment between consecutive samples.
- `tEnd`: The total length of the ray in texture coordinates.

**Data Structures:**
- `ray`: A 3D vector representing the direction and length of the ray in texture space.
- `samples`: A scalar representing the total number of samples to be taken.

**Steps:**
1. Calculate the ray vector in texture coordinates:
   $$ \text{ray} = \text{exitPoint} - \text{entryPoint} $$
2. Calculate the total number of samples required based on the ray length and sampling rate:
   $$ \text{samples} = \|\text{ray} \times \text{dim}\| \times \text{samplingRate} $$
   Note: The multiplication `ray * dim` scales the texture coordinate ray to world-like units if `dim` represents the volume resolution, or simply scales the vector. The source code implies `length(ray * dim)` computes the magnitude of the scaled vector.
3. Calculate the total length of the ray in texture coordinates:
   $$ t_{\text{End}} = \|\text{ray}\| $$
4. Calculate the step increment (distance between samples in parameter space):
   $$ t_{\text{Incr}} = \frac{t_{\text{End}}}{\text{samples}} $$
5. Set the start parameter to half a step from the entry point:
   $$ t_{\text{Start}} = 0.5 \times t_{\text{Incr}} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def entry_align(entry_point, exit_point, dim, sampling_rate):
    # 1. Calculate ray vector in texture coordinates
    ray = exit_point - entry_point
    
    # 2. Calculate total number of samples
    # Note: ray * dim scales the vector. length() computes the Euclidean norm.
    samples = length(ray * dim) * sampling_rate
    
    # 3. Calculate total ray length in texture coordinates
    t_end = length(ray)
    
    # 4. Calculate step increment
    t_incr = t_end / samples
    
    # 5. Set start parameter to half a step from entry
    t_start = 0.5 * t_incr
    
    return t_start, t_incr, t_end
```

**Parameters:**
- `samplingRate`: Determines the density of samples. Typical values depend on the desired visual quality and performance constraints. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Back Boundary Error:** While Entry Align corrects the estimation error at the entry point, the last sample may still under- or overestimate the contribution at the exit boundary, depending on how close the final sample is to the back side of the volume [S1, unofficial]. This is because the step size is fixed based on the entry alignment, and the exit point is not necessarily aligned with a sample midpoint.

## Subtopics

### [Entry Exit Align](856-entry-exit-align.md)
A variant that aligns both the first and last samples to minimize boundary artifacts at both entry and exit. It adjusts the step size per ray to ensure the last sample is half a step from the exit point. Recommended as best practice for zero performance impact and minimal errors.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. The calculation of `tStart` and `tIncr` is $O(1)$ per ray.
- **Memory Complexity:** $O(1)$ additional memory per ray for the parameters.
- **Strengths:** Eliminates the boundary estimation error at the entry point, which is a common artifact in naive fixed-step raycasting. Simple to implement.
- **Limitations:** Does not correct the boundary error at the exit point. The fixed step size may lead to suboptimal sampling density if the ray length varies significantly across pixels, though the step size is calculated per ray.
- **When to use:** When the entry boundary error is the primary concern and the exit boundary error is acceptable or can be mitigated by other means (e.g., [Entry Exit Align](856-entry-exit-align.md)).
- **When not to use:** When high accuracy at both boundaries is required; use [Entry Exit Align](856-entry-exit-align.md) instead.

## Background
Entry Align was introduced in the context of revisiting volume raycasting sampling strategies to address Riemann sum estimation errors [S1, unofficial]. It builds on the concept of midpoint Riemann sums, where sample points are shifted to the center of their respective integration intervals. The method is similar to techniques described in "Real-Time Volume Graphics" by Engel et al. [S1, unofficial].

## Sources
[S1] Volume Raycasting Sampling Revisited, https://osf.io/download/u9qnz/, fetched 2026-10-29 (unofficial)
