# Cubic Interpolation

Also called: 3D Cubic Interpolation, Tri-cubic Interpolation

## Overview
Cubic interpolation is a higher-order polynomial sampling technique used in volume rendering to approximate scalar values at arbitrary positions within a 3D grid, producing smoother results than linear methods [S1, S2, S3, unofficial]. It sets itself apart from its parent [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md) by referring to the general class of tri-cubic schemes that utilize a $4 \times 4 \times 4$ neighborhood of 64 voxels, whereas Mitchell-Netravali is a specific parameterized subset of this family [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume texture (voxel grid) with dimensions $R \times S \times T$, and a query position $\mathbf{p} = (x, y, z)$ in continuous space.
- Output: A single interpolated scalar value $V(\mathbf{p})$.

**Data Structures:**
- The volume is stored as a 3D array of floating-point values.
- Coordinates are typically normalized to $[0, 1]$ or mapped to integer indices $[0, R-1]$.

**Method:**
Tri-cubic interpolation is the tensor product of three 1D cubic interpolation kernels. For a query point, the algorithm identifies the $4 \times 4 \times 4$ block of 64 surrounding voxels that enclose the point.

1.  **Coordinate Mapping:** Convert the query position $\mathbf{p}$ into grid coordinates. If the volume has dimensions $R, S, T$ along axes $x, y, z$, and the query is in normalized texture coordinates $(u, v, w)$, compute the scaled coordinates:
    $$ U = u \times R, \quad V = v \times S, \quad W = w \times T $$
    [S4, unofficial]

2.  **Index Selection:** For each axis, determine the integer base index $i$ and the fractional part $f$.
    $$ i_x = \lfloor U \rfloor, \quad f_x = U - i_x $$
    $$ i_y = \lfloor V \rfloor, \quad f_y = V - i_y $$
    $$ i_z = \lfloor W \rfloor, \quad f_z = W - i_z $$
    [S4, unofficial]
    The base indices $i_x, i_y, i_z$ refer to the voxel closest to the query point in the negative direction. The 64 samples are taken from indices $i_x-1$ to $i_x+2$ (and similarly for $y, z$). Boundary conditions must handle indices outside $[0, R-1]$. [G] Common practice is clamping or wrapping, depending on the volume's semantic meaning.

3.  **1D Cubic Kernel Evaluation:** For each axis, compute the interpolation weights using a cubic kernel $K(f)$. The specific kernel function is not defined in the general "Cubic Interpolation" entry but is implied to be a standard cubic polynomial (e.g., Catmull-Rom or B-spline) or the general form from the Mitchell-Netravali family [S3, unofficial]. The weight for a neighbor at offset $k$ (where $k \in \{-1, 0, 1, 2\}$) is $K(f - k)$.
    [S3, unofficial] notes that cubic interpolation takes four interpolation points into account along a single dimension.

4.  **Trilinear Combination:** The final value is the sum of the 64 voxel values $V_{ijk}$ weighted by the product of the 1D kernel evaluations for each axis:
    $$ V(\mathbf{p}) = \sum_{k_x=-1}^{2} \sum_{k_y=-1}^{2} \sum_{k_z=-1}^{2} V_{i_x+k_x, i_y+k_y, i_z+k_z} \cdot K(f_x - k_x) \cdot K(f_y - k_y) \cdot K(f_z - k_z) $$

**Parameters:**
- The specific cubic kernel coefficients (e.g., $B, C$ in Mitchell-Netravali) are not fixed in the general definition [S3, unofficial]. They control the trade-off between smoothing and sharpening.

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The $4 \times 4 \times 4$ neighborhood extends 1 voxel beyond the query point in both directions. Indices outside the volume bounds must be handled. [G] Clamping to the nearest edge voxel is the most common approach for medical volumes.
- **Precision:** The increased number of multiplications (64 weights) can accumulate floating-point error, though typically negligible compared to the benefit of smoothness [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_interpolate(volume, u, v, w, R, S, T, kernel_func):
    """
    volume: 3D array [R, S, T]
    u, v, w: normalized coordinates in [0, 1]
    R, S, T: dimensions of the volume
    kernel_func: function f(fraction) returning the weight for a cubic kernel
    """
    # 1. Scale and offset
    U = u * R
    V = v * S
    W = w * T
    
    # 2. Integer and fractional parts
    ix = int(U)
    fx = U - ix
    iy = int(V)
    fy = V - iy
    iz = int(W)
    fz = W - iz
    
    # 3. Initialize sum
    result = 0.0
    
    # 4. Iterate over 4x4x4 neighborhood
    for dx in range(-1, 3):
        for dy in range(-1, 3):
            for dz in range(-1, 3):
                # Get voxel index
                x = ix + dx
                y = iy + dy
                z = iz + dz
                
                # Handle boundaries [G]
                if x < 0 or x >= R or y < 0 or y >= S or z < 0 or z >= T:
                    weight = 0.0
                else:
                    # Evaluate kernel weights for each dimension
                    wx = kernel_func(fx - dx)
                    wy = kernel_func(fy - dy)
                    wz = kernel_func(fz - dz)
                    weight = wx * wy * wz
                
                result += volume[x, y, z] * weight
                
    return result
```

## Subtopics

### Quasi-Cubic Sampling
A texture sampling technique that applies a quasi-cubic filter kernel to 3D volume textures to reduce aliasing and improve interpolation quality compared to standard linear/trilinear filtering [S4, unofficial]. It differs from full tri-cubic interpolation by using a specific simplified kernel application on normalized coordinates rather than a full 64-voxel convolution [S4, unofficial].
**Steps:**
1. Scale the normalized texture coordinate $u$ by the volume dimension $R$ and add 0.5 to center the texels: $U_{scaled} = u \times R + 0.5$ [S4, unofficial].
2. Extract the fractional part $F = fract(U_{scaled})$ and the integer part. Apply the quasi-cubic kernel to the fractional part: $U_{filtered} = floor(U_{scaled}) + F \times F \times (3.0 - 2.0 \times F)$ [S4, unofficial]. Note: This formula resembles a smoothstep function applied to the fractional part, often used in hardware-accelerated "quasi-cubic" or "bicubic" approximations.
3. Sample the volume texture using the filtered coordinate normalized back to $[0, 1]$: $sample = texture(volume, (U_{filtered} - 0.5) / R)$ [S4, unofficial].
**When to choose:** When hardware support for true tri-cubic filtering is limited or when a balance between the speed of trilinear and the quality of full cubic is needed, specifically for reducing aliasing artifacts [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Considerably high computational costs. In 3D, it requires sampling 64 voxels ($2^6$) per query point [S1, S2, unofficial]. It requires two more multiplications per dimension compared to linear interpolation, leading to a considerable amount of instructions [S3, unofficial].
- **Memory Complexity:** Requires access to a $4 \times 4 \times 4$ neighborhood, increasing cache pressure compared to trilinear (8 voxels) [S2, unofficial].
- **Performance:** Produces smoother results than linear interpolation but increases render time significantly [S1, S2, unofficial].
- **Strengths:** Superior image quality, especially at large magnification factors, preventing the "3D cross" (diamond) artifacts seen in trilinear interpolation [S3, unofficial].
- **Limitations:** High computational cost makes it less suitable for real-time applications on standard hardware without optimization or approximation [S3, unofficial].
- **When to use:** Offline rendering or high-quality visualization where smoothness is critical and performance is secondary [S3, unofficial].
- **When not to use:** Real-time applications with strict latency constraints where trilinear provides sufficient quality [S3, unofficial].

## Background
Cubic interpolation in volume rendering evolved as a response to the blocky artifacts and discontinuities of nearest-neighbor and trilinear interpolation. Trilinear interpolation, while efficient, suffers from visible artifacts at high magnification, such as 3D crosses or diamonds [S3, unofficial]. Higher-order methods like cubic convolution were developed to provide $C^1$ continuity (smooth first derivatives), reducing high-frequency leakage artifacts [S3, unofficial]. The general tri-cubic approach using 64 samples is a standard technique in medical volume rendering and scientific visualization to achieve photorealistic smoothness [S2, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S3] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S4] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
