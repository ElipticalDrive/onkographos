# Cubic Interpolation (Mitchell-Netravali)

Also called: Mitchell-Netravali, Mitchell-Netravali Cubic Filter

## Overview
Mitchell-Netravali cubic interpolation is a family of C0 and C1 continuous cubic filters defined by parameters B and C, used to reconstruct volume data while preventing high-frequency leakage artifacts [S1]. It distinguishes itself from simpler schemes by offering a tunable trade-off between blurring and ringing, and from fixed B-splines by allowing the generation of Catmull-Rom splines [S1]. It builds on [Fast tri-cubic interpolation](848-fast-tri-cubic-interpolation.md) by providing the specific kernel weights for the convolution sum [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume texture $V$, a sampling position $P$, and tuning parameters $B$ and $C$ [S1].
- Output: A single interpolated scalar value at $P$ [G].
- Parameters: Tuning parameters $B$ and $C$ [S1].

**Data Structures:**
- The kernel $k(x)$ is a scalar function [S1].
- The volume $V$ is accessed via integer voxel coordinates [G].

**Method:**
1. For each sample point $P$ along the ray, compute the fractional offset $x$ from the nearest voxel center [G].
2. Compute the kernel weight $k(x)$ using the Mitchell-Netravali formula [S1].
3. Fetch the 64 nearest voxel values in 3D (4 samples per dimension) [S1].
4. Compute the weighted sum of these 64 samples [G].
5. Clamp the result to the valid input range to prevent out-of-bounds values [S1].

**Kernel Formula:**
The kernel $k(x)$ is defined piecewise based on $|x|$:

$$k(x) = \frac{1}{6} \begin{cases} (12 - 9B - 6C)|x^3| + (-18 + 12B + 6C)|x^2| + (6 - 2B), & \text{if } |x| < 1; \\ (-B - 6C)|x^3| + (6B + 30C)|x^2| + (-12B - 48C)|x| + (8B + 24C), & \text{if } 1 \le |x| < 2; \\ 0, & \text{otherwise.} \end{cases}$$ [S1]

**Symbol Definitions:**
- $x$: The distance from the sample point to the voxel center, normalized such that voxel centers are at integers [S1].
- $B, C$: Tuning parameters that trade off aliasing effects (blurring, ringing, anisotropy) [S1].
- $|x|$: Absolute value of $x$ [G].

**Parameters:**
- $B=1.0, C=0.0$: Generates Cubic B-splines [S1].
- $B=0.0, C=0.5$: Generates Catmull-Rom splines [S1].
- Typical values trade off aliasing effects (blurring, ringing, anisotropy) [S1].

**Pitfalls:**
- General cubic interpolation can produce values outside the range of the inputs; these must be clamped [S1].
- The filter satisfies $\sum_{n=-\infty}^{\infty} h(x-n) = 1$ to prevent sample frequency ripple [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def mitchell_netravali_kernel(x, B, C):
    abs_x = abs(x)
    if abs_x < 1.0:
        return ((12.0 - 9.0 * B - 6.0 * C) * abs_x**3 +
                (-18.0 + 12.0 * B + 6.0 * C) * abs_x**2 +
                (6.0 - 2.0 * B)) / 6.0
    elif abs_x < 2.0:
        return ((-B - 6.0 * C) * abs_x**3 +
                (6.0 * B + 30.0 * C) * abs_x**2 +
                (-12.0 * B - 48.0 * C) * abs_x +
                (8.0 * B + 24.0 * C)) / 6.0
    else:
        return 0.0

def sample_volume_mitchell(V, x_frac, y_frac, z_frac, B, C):
    # x_frac, y_frac, z_frac are fractional coordinates in [0, 1) relative to base voxel
    base_x = math.floor(x_frac)
    base_y = math.floor(y_frac)
    base_z = math.floor(z_frac)
    
    weighted_sum = 0.0
    total_weight = 0.0
    
    for dx in range(-1, 3):
        for dy in range(-1, 3):
            for dz in range(-1, 3):
                dist_x = x_frac - dx
                dist_y = y_frac - dy
                dist_z = z_frac - dz
                
                w_x = mitchell_netravali_kernel(dist_x, B, C)
                w_y = mitchell_netravali_kernel(dist_y, B, C)
                w_z = mitchell_netravali_kernel(dist_z, B, C)
                
                weight = w_x * w_y * w_z
                voxel_val = V[base_x + dx, base_y + dy, base_z + dz]
                
                weighted_sum += weight * voxel_val
                total_weight += weight
                
    result = weighted_sum / total_weight if total_weight > 0 else 0.0
    return max(0.0, min(1.0, result))
```

## Subtopics

### Cubic Interpolation
A higher-order polynomial interpolation scheme for smoother results than linear interpolation, requiring more samples [S2]. In 3D, it requires sampling 64 voxels ($2^4$) [S2]. It produces smoother results than linear interpolation but increases render time [S2].

## Operating characteristics
**Time and Memory Complexity:**
- In 3D, cubic interpolation requires sampling 64 nearest samples [S1].
- Spatial extent is $4N$ samples, where $N$ is the dimension of the input (i.e., 4, 16, or 64 for 1D, 2D, or 3D cases) [S1].
- As complexity increases, quality increases, but so does computation [S1].

**Strengths and Limitations:**
- Strengths: C0 and C1 continuity prevents high-frequency leakage artifacts [S1].
- Tunable parameters allow trade-offs between blurring, ringing, and anisotropy [S1].
- Limited spatial extent [S1].
- Limitations: Higher computational cost than linear or nearest-neighbor [S1].
- Can produce values outside the input range, requiring clamping [S1].

**When to Use:**
- Use when smoother results than linear interpolation are needed and computational resources allow for 64 samples per point [S2].
- Use when specific artifact characteristics (ringing vs. blurring) need to be tuned via B and C parameters [S1].

## Background
Originated from Mitchell and Netravali's work on reconstruction filters [Mitchell 88], as cited in [S1]. The method is designed to provide C0 and C1 continuity to prevent high-frequency leakage artifacts [S1]. It is used in volume rendering for publication-quality imagery [S1].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27
[S2] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
