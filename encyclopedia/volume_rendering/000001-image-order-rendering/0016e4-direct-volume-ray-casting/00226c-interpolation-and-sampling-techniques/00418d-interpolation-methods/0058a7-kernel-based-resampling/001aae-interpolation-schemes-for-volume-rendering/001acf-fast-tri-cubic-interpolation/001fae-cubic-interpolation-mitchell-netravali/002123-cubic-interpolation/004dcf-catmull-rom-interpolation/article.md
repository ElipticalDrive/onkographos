# Catmull-Rom Interpolation
Also called: Cardinal Spline, Cardinal Spline / Catmull-Rom Filter, Cardinal spline interpolation, Catmull-Rom spline, Cubic filter (specifically Catmull-Rom), Cubic-spline interpolation function, Second-order interpolation filter, Third-order error minimizing cubic filter

## Overview
Catmull-Rom interpolation is a specific interpolating cubic-spline scheme used in volume rendering to reconstruct continuous scalar fields from discrete voxel data [S1, unofficial]. It solves the problem of banding artifacts inherent in linear interpolation by providing a continuous first derivative, while uniquely satisfying the condition for accurate straight-line reconstruction among all interpolating cubic splines [S1, S2, unofficial]. It is distinguished from its sibling [General Cubic Spline Interpolation](15870-general-cubic-spline-interpolation.md) by its fixed parameter values ($B=0, C=1/2$ or $a=-0.5$), which minimize third-order error for discrete functions [S2, unofficial][S3, book].

## How it works
**Inputs and Outputs**
- Input: A 3D volume grid of scalar values $V_{i,j,k}$ and a query point $(x, y, z)$ in continuous space [G].
- Output: The interpolated scalar value $f(x, y, z)$ [G].

**Data Structures**
- The volume is stored as a 3D array [G].
- The filter kernel is defined by a 1D piecewise polynomial function $h(u)$ with support width 2 (spanning two grid intervals) [S2, unofficial].

**Formula**
The 1D filter function $h(u)$ depends on the distance $u$ from the sample location to the nearest grid point [S2, unofficial]. It is defined piecewise as [S1, unofficial]:

$$
h(u) = \begin{cases} 
h(-u) & u < 0 \\
\frac{5}{2}u^3 - \frac{5}{2}u^2 + 1 & 0 \le u < 1 \\
-\frac{1}{2}u^3 + \frac{5}{2}u^2 - 4u + 2 & 1 \le u < 2 \\
0 & \text{otherwise}
\end{cases}
$$

This specific form corresponds to the general cubic spline family with parameters $B=0$ and $C=1/2$ [S1, unofficial]. The parameterized form with $a = -0.5$ yields the Catmull-Rom spline, which interpolates a discrete function with the lowest third-order error [S2, unofficial][S3, book].

**Steps**
1. **Coordinate Transformation**: For the query point $(x, y, z)$, determine the integer grid indices $(i, j, k)$ such that $i \le x < i+1$, $j \le y < j+1$, and $k \le z < k+1$ [G].
2. **Fractional Offsets**: Compute the fractional distances $u = x - i$, $v = y - j$, and $w = z - k$ [G].
3. **Kernel Evaluation**: Evaluate the 1D filter function $h(u)$, $h(v)$, and $h(w)$ using the piecewise formula above [S2, unofficial]. Note that $h(u)$ requires values from grid points at indices $i-1, i, i+1, i+2$ to compute the contribution for the interval $[i, i+1]$ [S2, unofficial].
4. **Separable 3D Interpolation**: Since the 3D filter is separable, $h(u, v, w) = h(u)h(v)h(w)$, the interpolation can be computed as a 3-stage nested loop [S2, unofficial][S3, book].
    - First, interpolate along the x-axis for each $(j, k)$ plane to get intermediate values [S2, unofficial][S3, book].
    - Second, interpolate along the y-axis for each $k$ line [S2, unofficial][S3, book].
    - Third, interpolate along the z-axis to get the final scalar value [S2, unofficial][S3, book].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def catmull_rom_1d(u):
    """Computes the 1D Catmull-Rom kernel value for distance u >= 0."""
    if u < 0:
        u = -u
    if u < 1:
        return 2.5 * u**3 - 2.5 * u**2 + 1.0
    elif u < 2:
        return -0.5 * u**3 + 2.5 * u**2 - 4.0 * u + 2.0
    return 0.0

def interpolate_volume(volume, x, y, z):
    """Interpolates a value from a 3D volume grid using Catmull-Rom interpolation."""
    ix, iy, iz = int(x), int(y), int(z)
    u, v, w = x - ix, y - iy, z - iz
    
    # Precompute 1D weights for each dimension
    # Offsets: -1, 0, 1, 2 relative to the integer grid point
    wx = [catmull_rom_1d(1.0 + u), catmull_rom_1d(u), catmull_rom_1d(1.0 - u), catmull_rom_1d(2.0 - u)]
    wy = [catmull_rom_1d(1.0 + v), catmull_rom_1d(v), catmull_rom_1d(1.0 - v), catmull_rom_1d(2.0 - v)]
    wz = [catmull_rom_1d(1.0 + w), catmull_rom_1d(w), catmull_rom_1d(1.0 - w), catmull_rom_1d(2.0 - w)]
    
    W, H, D = len(volume[0][0]), len(volume[0]), len(volume)
    
    # Stage 1: Interpolate along x-axis for each (j, k) plane
    temp = [[0.0] * D for _ in range(H)]
    for j in range(H):
        for k in range(D):
            val = 0.0
            for i in range(4):
                gx = ix + i - 1
                if 0 <= gx < W:
                    val += wx[i] * volume[j][k][gx]
            temp[j][k] = val
            
    # Stage 2: Interpolate along y-axis for each k line
    temp2 = [0.0] * D
    for k in range(D):
        val = 0.0
        for j in range(4):
            gy = iy + j - 1
            if 0 <= gy < H:
                val += wy[j] * temp[gy][k]
        temp2[k] = val
        
    # Stage 3: Interpolate along z-axis
    result = 0.0
    for k in range(4):
        gz = iz + k - 1
        if 0 <= gz < D:
            result += wz[k] * temp2[gz]
    return result
```

**Edge Cases and Pitfalls**
- The second-order derivative is not continuous, which may cause subtle visual artifacts in curvature-sensitive applications [S1, unofficial].
- Boundary conditions are not defined by the kernel itself; implementation must handle grid edges (e.g., clamping, mirroring, or extrapolation) [G].
- The support width is 2 grid intervals, requiring 4 samples per axis [S2, unofficial][S3, book].

## Subtopics

### [General Cubic Spline Interpolation](15870-general-cubic-spline-interpolation.md)
A broader family of cubic interpolators with tunable parameters $B$ and $C$. Choose this when you need to balance between sharpness and blurring or when specific frequency response characteristics are required. Catmull-Rom is the specific case where $B=0, C=1/2$.

## Operating characteristics
- **Time Complexity**: $O(1)$ per sample, but with a high constant factor due to the 64-sample support (4x4x4 grid points) and cubic polynomial evaluations [S2, unofficial][S3, book].
- **Memory Complexity**: $O(1)$ auxiliary memory per sample [G].
- **Performance**: Higher filter quality than linear interpolation, but at the price of wider spatial extent and larger computational effort [S2, unofficial][S3, book].
- **Strengths**: Continuous first derivative reduces banding artifacts compared to linear filters; accurately reconstructs straight lines; minimizes third-order error for discrete functions [S1, S2, unofficial][S3, book].
- **Limitations**: Discontinuous second derivative; computationally expensive due to large support; potential for ringing if not windowed (though less severe than sinc) [S2, unofficial][S3, book].
- **When to use**: When smooth gradients are required and straight-line fidelity is important, and computational resources allow for cubic filtering [S2, unofficial][S3, book].
- **When not to use**: For real-time applications with strict performance constraints where linear interpolation suffices, or when ringing artifacts are unacceptable without additional windowing [S2, unofficial][S3, book].

## Background
The Catmull-Rom spline is a specific case of the cubic-spline interpolation function with parameters $B=0$ and $C=1/2$ [S1, unofficial]. It is the only interpolating cubic-spline function able to reconstruct a straight line [S1, unofficial]. In volume rendering, it is cited as a standard second-order interpolation filter that yields a function with a continuous first derivative, reducing banding artifacts [S2, unofficial].

## Sources
[S1] Iso-Surface Volume Rendering : speed and accuracy for medical applications, https://doi.org/10.3990/1.9789036513982, fetched 2026-10-26 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
