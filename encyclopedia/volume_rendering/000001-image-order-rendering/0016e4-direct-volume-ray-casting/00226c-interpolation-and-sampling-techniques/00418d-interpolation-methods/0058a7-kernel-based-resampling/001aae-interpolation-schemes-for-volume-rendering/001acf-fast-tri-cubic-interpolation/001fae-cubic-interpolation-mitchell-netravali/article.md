# Cubic Interpolation (Mitchell-Netravali)
Also called: Mitchell-Netravali, Mitchell-Netravali Cubic Filter

## Overview
Mitchell-Netravali cubic interpolation is a kernel-based resampling method that reconstructs a scalar value from a volume using a piecewise cubic filter defined by parameters $B$ and $C$ [S1, unofficial]. It distinguishes itself from its siblings by providing $C^0$ and $C^1$ continuity to prevent high-frequency leakage artifacts, while allowing tunable trade-offs between blurring and ringing [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume texture $V$, a query point $\mathbf{p} = (x, y, z)$ in voxel coordinates [G].
- Output: A scalar value $f(\mathbf{p})$ [G].

**Data Structures:**
- The volume $V$ is stored as a 3D array of floating-point values [G].
- The kernel $k(x)$ is defined by two real-valued parameters $B$ and $C$ [S1, unofficial].

**Algorithm:**
1. **Determine Sampling Grid:** For each dimension $d \in \{x, y, z\}$, identify the integer grid points surrounding the query coordinate $p_d$. The kernel support extends to $|x| < 2$, requiring 4 samples per dimension [S1, unofficial]. Let the indices be $i-1, i, i+1, i+2$ where $i = \lfloor p_d \rfloor$ [G].
2. **Compute Relative Distances:** For each sample index $j \in \{-1, 0, 1, 2\}$, compute the relative distance $u = p_d - (i + j)$ [G].
3. **Evaluate Kernel:** Compute the weight $w = k(u)$ using the Mitchell-Netravali formula [S1, unofficial].
4. **Convolve:** Sum the weighted values from all $4 \times 4 \times 4 = 64$ neighboring voxels [S1, unofficial].
5. **Clamp:** Clamp the final result to the input range to prevent values outside the valid data range [S1, unofficial].

**Kernel Formula:**
The 1D kernel $k(x)$ is defined piecewise based on $|x|$:

$$
k(x) = \frac{1}{6} \begin{cases} 
(12 - 9B - 6C)|x|^3 + (-18 + 12B + 6C)|x|^2 + (6 - 2B), & \text{if } |x| < 1; \\
(-B - 6C)|x|^3 + (6B + 30C)|x|^2 + (-12B - 48C)|x| + (8B + 24C), & \text{if } 1 \le |x| < 2; \\
0, & \text{otherwise.}
\end{cases}
$$

Note: The source text uses $|x^3|$ and $|x^2|$. Since $|x|^3 = |x^3|$ and $|x|^2 = x^2$, the absolute value on the cubic term is redundant for real numbers but preserves the symmetry logic. The standard implementation uses $|x|$ for the linear term in the second case [G].

**Parameters:**
- $B, C$: Tuning parameters that trade off aliasing effects (blurring, ringing, anisotropy) [S1, unofficial].
- Typical values:
    - Cubic B-spline: $B=1.0, C=0.0$ [S1, unofficial].
    - Catmull-Rom: $B=0.0, C=0.5$ [S1, unofficial].
- Constraint: The filter satisfies $\sum_{n=-\infty}^{\infty} k(x-n) = 1$ to prevent sample frequency ripple [S1, unofficial]. (The source denotes the filter function as $h(x)$ in this constraint, but $k(x)$ is used for the kernel definition [S1, unofficial].)

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def clamp(val, min_val, max_val):
    return max(min_val, min(max_val, val))

def mitchell_netravali_kernel(x, B, C):
    ax = abs(x)
    ax2 = ax * ax
    ax3 = ax2 * ax
    
    if ax < 1.0:
        c3 = 12.0 - 9.0 * B - 6.0 * C
        c2 = -18.0 + 12.0 * B + 6.0 * C
        c0 = 6.0 - 2.0 * B
        return (c3 * ax3 + c2 * ax2 + c0) / 6.0
    elif ax < 2.0:
        c3 = -B - 6.0 * C
        c2 = 6.0 * B + 30.0 * C
        c1 = -12.0 * B - 48.0 * C
        c0 = 8.0 * B + 24.0 * C
        return (c3 * ax3 + c2 * ax2 + c1 * ax + c0) / 6.0
    else:
        return 0.0

def sample_volume_mitchell(volume, p, B, C, min_val, max_val):
    result = 0.0
    ix = math.floor(p[0])
    iy = math.floor(p[1])
    iz = math.floor(p[2])
    
    for dx in range(-1, 3):
        for dy in range(-1, 3):
            for dz in range(-1, 3):
                vx = ix + dx
                vy = iy + dy
                vz = iz + dz
                
                if not volume.is_valid(vx, vy, vz):
                    continue
                
                ux = p[0] - (ix + dx)
                uy = p[1] - (iy + dy)
                uz = p[2] - (iz + dz)
                
                wx = mitchell_netravali_kernel(ux, B, C)
                wy = mitchell_netravali_kernel(uy, B, C)
                wz = mitchell_netravali_kernel(uz, B, C)
                
                w = wx * wy * wz
                result += w * volume.get(vx, vy, vz)
                
    return clamp(result, min_val, max_val)
```

## Subtopics

### [Cubic Interpolation](2438-cubic-interpolation.md)
A general higher-order polynomial reconstruction method sampling 64 voxels. It distinguishes itself by utilizing a full cubic kernel to minimize high-frequency leakage, at the cost of high computational complexity.

## Operating characteristics
- **Complexity:** Time complexity is $O(1)$ per sample but with a high constant factor. In 3D, it requires 64 texture fetches per output sample [S1, unofficial].
- **Memory:** No additional memory beyond the volume texture [G].
- **Strengths:** $C^1$ continuity reduces artifacts compared to linear interpolation; tunable parameters allow balancing between sharpness and smoothness [S1, unofficial].
- **Limitations:** High computational cost due to 64 samples; potential for values outside the input range requiring clamping [S1, unofficial].
- **When to use:** When high-quality, smooth reconstruction is required and performance allows for 64 samples per ray step [G].

## Background
The Mitchell filter was introduced by Mitchell in 1988 [S1, unofficial]. It is a family of cubic filters designed to provide both $C^0$ and $C^1$ continuity, preventing high-frequency leakage artifacts common in lower-order interpolations [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
