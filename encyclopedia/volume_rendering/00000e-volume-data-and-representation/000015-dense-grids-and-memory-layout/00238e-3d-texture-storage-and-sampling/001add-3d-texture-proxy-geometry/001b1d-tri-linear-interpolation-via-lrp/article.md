# Tri-linear Interpolation via LRP

Also called: Tri-linear Interpolation, LRP-based Interpolation

## Overview
Tri-linear interpolation via LRP is a GPU-accelerated method for estimating scalar values at arbitrary floating-point coordinates within a 3D volume grid by interpolating the eight surrounding voxel values [S2, S3, unofficial]. It solves the problem of sampling continuous volume data from discrete voxel grids, providing smooth rendering compared to direct voxel rendering which exhibits less structural detail [S1, S2, unofficial]. It is distinguished from other interpolation methods by its specific implementation using the GPU's LRP (linear interpolation) instruction, leveraging hardware rasterization to interpolate texture coordinates "for free" between vertices, thereby optimizing memory bandwidth usage [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture (or array) with dimensions $W \times H \times D$, and a query point $(x, y, z)$ in continuous space.
*   **Output:** A single interpolated scalar value $S_X$ at $(x, y, z)$.

**Data Structures:**
*   **Volume Data:** A 3D array or 3D texture storing voxel values. Indices are integer coordinates $(i, j, k)$ where $0 \le i < W$, $0 \le j < H$, $0 \le k < D$.
*   **Texture Coordinates:** 3D coordinates $(u, v, w)$ in the range $[0, 1]$ representing the position within the volume.
*   **Blending Factors:** Three fractional values $(\alpha, \beta, \gamma)$ representing the relative distance to the corner voxels along each axis.

**Numbered Steps:**

1.  **Coordinate Conversion:** Convert the continuous query point $(x, y, z)$ to integer floor and ceiling coordinates:
    $$
    \begin{aligned}
    x_f &= \lfloor x \rfloor, & x_c &= \lceil x \rceil \\
    y_f &= \lfloor y \rfloor, & y_c &= \lceil y \rceil \\
    z_f &= \lfloor z \rfloor, & z_c &= \lceil z \rceil
    \end{aligned}
    $$
    [S2, unofficial]

2.  **Boundary Clamping:** Ensure indices are within valid bounds. If any ceiling coordinate exceeds the volume dimension (e.g., $x_c \ge W$), clamp it to the maximum index ($W - 1$). This prevents out-of-bounds array access. [S2, unofficial]

3.  **Voxel Fetching:** Retrieve the eight corner voxel values surrounding the query point:
    $$
    \begin{aligned}
    S_{000} &= \text{Volume}[x_f, y_f, z_f] \\
    S_{001} &= \text{Volume}[x_f, y_f, z_c] \\
    S_{010} &= \text{Volume}[x_f, y_c, z_f] \\
    S_{011} &= \text{Volume}[x_f, y_c, z_c] \\
    S_{100} &= \text{Volume}[x_c, y_f, z_f] \\
    S_{101} &= \text{Volume}[x_c, y_f, z_c] \\
    S_{110} &= \text{Volume}[x_c, y_c, z_f] \\
    S_{111} &= \text{Volume}[x_c, y_c, z_c]
    \end{aligned}
    $$
    [S2, unofficial]

4.  **Blending Factor Calculation:** Compute the fractional offsets (blending factors) for each axis:
    $$
    \alpha = x - x_f, \quad \beta = y - y_f, \quad \gamma = z - z_f
    $$
    [S2, unofficial]
    
    *GPU-Specific Implementation (LRP):* In the context of the LRP instruction implementation, blending factors are derived from 3D texture coordinates $(u, v, w)$ of the resampling point. The texture coordinates are multiplied by the original texture resolution $(W, H, D)$. The fractional parts of these products serve as the blending factors:
    $$
    \alpha = \text{frac}(u \cdot W), \quad \beta = \text{frac}(v \cdot H), \quad \gamma = \text{frac}(w \cdot D)
    $$
    [S3, unofficial]

5.  **Interpolation:** Calculate the final value $S_X$ using the trilinear formula:
    $$
    \begin{aligned}
    S_X = & (1-\alpha)(1-\beta)(1-\gamma)S_{000} + \alpha(1-\beta)(1-\gamma)S_{100} \\
          & + (1-\alpha)\beta(1-\gamma)S_{010} + \alpha\beta(1-\gamma)S_{110} \\
          & + (1-\alpha)(1-\beta)\gamma S_{001} + \alpha(1-\beta)\gamma S_{101} \\
          & + (1-\alpha)\beta\gamma S_{011} + \alpha\beta\gamma S_{111}
    \end{aligned}
    $$
    [S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tri_linear_interpolation(volume, x, y, z, width, height, depth):
    """
    Performs tri-linear interpolation on a 3D volume.
    
    Args:
        volume: 3D array or texture object supporting indexing.
        x, y, z: Continuous coordinates to sample.
        width, height, depth: Dimensions of the volume.
        
    Returns:
        Interpolated scalar value.
    """
    # Step 1: Compute floor and ceiling coordinates
    xf = int(x)
    xc = xf + 1
    yf = int(y)
    yc = yf + 1
    zf = int(z)
    zc = zf + 1
    
    # Step 2: Boundary Clamping
    # Clamp ceiling coordinates to max index to prevent out-of-bounds access
    if xc >= width: xc = width - 1
    if yc >= height: yc = height - 1
    if zc >= depth: zc = depth - 1
    
    # Step 3: Fetch 8 corner voxel values
    S000 = volume[xf, yf, zf]
    S001 = volume[xf, yf, zc]
    S010 = volume[xf, yc, zf]
    S011 = volume[xf, yc, zc]
    S100 = volume[xc, yf, zf]
    S101 = volume[xc, yf, zc]
    S110 = volume[xc, yc, zf]
    S111 = volume[xc, yc, zc]
    
    # Step 4: Calculate blending factors (fractional parts)
    alpha = x - xf
    beta = y - yf
    gamma = z - zf
    
    # Step 5: Trilinear Interpolation Formula
    # Interpolate along X axis first
    S00 = (1 - alpha) * S000 + alpha * S100
    S01 = (1 - alpha) * S001 + alpha * S101
    S10 = (1 - alpha) * S010 + alpha * S110
    S11 = (1 - alpha) * S011 + alpha * S111
    
    # Interpolate along Y axis
    S0 = (1 - beta) * S00 + beta * S10
    S1 = (1 - beta) * S01 + beta * S11
    
    # Interpolate along Z axis
    result = (1 - gamma) * S0 + gamma * S1
    
    return result

def tri_linear_interpolation_lrp(volume, u, v, w, width, height, depth):
    """
    GPU-style tri-linear interpolation using LRP instruction logic.
    Assumes u, v, w are texture coordinates in [0, 1].
    """
    # Step 4 (LRP variant): Calculate blending factors from texture coordinates
    alpha = (u * width) % 1.0
    beta = (v * height) % 1.0
    gamma = (w * depth) % 1.0
    
    # Convert to integer indices for fetching
    # Note: In hardware, these indices are pre-computed and interpolated during rasterization
    xf = int(u * width)
    xc = min(xf + 1, width - 1)
    yf = int(v * height)
    yc = min(yf + 1, height - 1)
    zf = int(w * depth)
    zc = min(zf + 1, depth - 1)
    
    # Step 3: Fetch 8 corner voxel values
    S000 = volume[xf, yf, zf]
    S001 = volume[xf, yf, zc]
    S010 = volume[xf, yc, zf]
    S011 = volume[xf, yc, zc]
    S100 = volume[xc, yf, zf]
    S101 = volume[xc, yf, zc]
    S110 = volume[xc, yc, zf]
    S111 = volume[xc, yc, zc]
    
    # Step 5: Trilinear Interpolation Formula
    S00 = (1 - alpha) * S000 + alpha * S100
    S01 = (1 - alpha) * S001 + alpha * S101
    S10 = (1 - alpha) * S010 + alpha * S110
    S11 = (1 - alpha) * S011 + alpha * S111
    
    S0 = (1 - beta) * S00 + beta * S10
    S1 = (1 - beta) * S01 + beta * S11
    
    result = (1 - gamma) * S0 + gamma * S1
    
    return result
```

**Parameters:**
*   `tex_coord`: 3D texture coordinates of the resampling point, range $[0, 1]$ [S3, unofficial].
*   `resolution`: Original dimensions of the 3D texture $(W, H, D)$ [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Bandwidth:** Trilinear interpolation requires at least eight memory reads per sample point. For real-time rendering, this creates a high demand for memory bandwidth. If memory latency is high, rendering speed can be severely slowed down [S1, unofficial].
*   **Boundary Handling:** If the ceiling coordinate exceeds the volume dimension, it must be clamped to the maximum index to prevent out-of-bounds access [S2, unofficial].
*   **Precision:** The calculation of blending factors using fractional parts of multiplied coordinates relies on floating-point precision. [G] Ensure that texture coordinates are normalized correctly to avoid off-by-one errors at boundaries.

## Subtopics

### [Coarsest-Level Lerp](2770-coarsest-level-lerp.md)
A method for computing the value of a coarser-side octant vertex by performing trilinear interpolation on the coarsest level involved for any of the inputs [2770-coarsest-level-lerp.md]. It is chosen when working with multi-resolution volume data (mipmaps) to ensure consistency across levels.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point, as it involves a fixed number of arithmetic operations and memory accesses regardless of volume size.
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample, but requires access to 8 neighboring voxels.
*   **Measured Performance:** Requires at least eight million memory reads per frame for typical resolutions, making memory bandwidth a critical bottleneck [S1, unofficial].
*   **Strengths:** Provides smooth rendering of volumetric data, preserving structural details better than direct voxel rendering [S1, S2, unofficial]. Hardware-accelerated via LRP instructions and texture sampling units [S3, unofficial].
*   **Limitations:** High memory bandwidth consumption can limit frame rates on hardware with limited memory throughput [S1, unofficial].
*   **When to Use:** When smooth visualization of volume data is required and memory bandwidth is sufficient.
*   **When Not to Use:** When memory bandwidth is severely constrained and performance is critical, or when the volume data is sparse and sparse data structures (like Octrees) are more appropriate.

## Background
Tri-linear interpolation is the primary interpolation method for volume rendering, providing smooth rendering of volumetric data [S1, unofficial]. It is a method of multivariate interpolation on a 3-D regular grid that approximates the value of a function at an intermediate point linearly, using function data on the lattice points [S2, unofficial]. The GPU-based implementation using the LRP instruction was described by Viola et al. [2002] in the context of hardware-accelerated frequency domain volume rendering [S3, unofficial]. It is widely used in real-time volume rendering systems to handle the interpolation of millions of sample points per frame [S1, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S3] PDF Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S4] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
