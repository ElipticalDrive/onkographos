# Cubic Interpolation

Also called: 3D Cubic Interpolation, Tri-cubic Interpolation

## Overview
Cubic interpolation is a higher-order polynomial sampling technique used in volume rendering to approximate scalar field values at arbitrary locations with greater smoothness than linear methods [S1, S2, S3, unofficial]. It solves the artifact-prone nature of nearest-neighbor and linear interpolation by utilizing a wider support kernel, resulting in visually smoother surfaces and reduced high-frequency leakage [S1, S2, unofficial]. Unlike its parent [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md), which defines a general family of filters parameterized by $B$ and $C$, this article focuses on the specific implementation characteristics of standard tri-cubic sampling, particularly the computationally optimized "on-the-fly" approach and the specific Catmull-Rom variant often used as the default cubic filter.

## How it works
### Inputs and Outputs
- **Input**: A 3D volume texture (voxel grid) with dimensions $R \times S \times T$, and a query point $\mathbf{p} = (x, y, z)$ in texture space.
- **Output**: A single interpolated scalar value $V(\mathbf{p})$.

### Data Structures
The volume is typically stored as a 3D array or a 3D texture object. For hardware-accelerated sampling, the data is often packed into a contiguous memory block or GPU texture format (e.g., `GL_RGBA32F` or similar, depending on precision requirements). The coordinates are normalized to $[0, 1]$ for texture lookup or kept in voxel indices for software implementation.

### Number of Samples
In 3D, standard cubic interpolation requires sampling $4 \times 4 \times 4 = 64$ neighboring voxels [S1, S2, S5, unofficial]. This is derived from the fact that a 1D cubic kernel spans 4 points (2 on each side of the query point), and the 3D kernel is the separable product of three 1D kernels [S3, unofficial].

### The Tri-Cubic Filter Function
The 3D tri-cubic filter $h(u, v, w)$ is separable:
$$ h(u, v, w) = h(u)h(v)h(w) $$
where $u, v, w$ are the distances from the query point to the grid points along each axis [S6, unofficial].

The most common specific cubic filter used in this context is the **Catmull-Rom spline**, which is a special case of the cubic convolution filter [S6, unofficial]. The 1D filter function $h(u)$ for Catmull-Rom is defined piecewise based on the distance $u$ from the nearest grid point:

$$
h(u) = 
\begin{cases} 
2u^3 - 3u^2 + 1 & 0 \le u < 1 \\
-2u^3 + 3u^2 & 1 \le u \le 2 \\
0 & u > 2 
\end{cases}
$$

This corresponds to the general cubic convolution formula with parameter $a = -0.5$ [S6, unofficial]:
$$
h(u) = 
\begin{cases} 
(a+2)u^3 - (a+3)u^2 + 1 & 0 \le u < 1 \\
au^3 - 5au^2 + 8au - 4a + 1 & 1 \le u \le 2 \\
0 & u > 2 
\end{cases}
$$
With $a = -0.5$, the coefficients become $1.5$ and $-0.5$ respectively, yielding the Catmull-Rom form above [S6, unofficial].

### Implementation: On-the-Fly Tri-Cubic Filtering
A common optimization for implementing tri-cubic interpolation on hardware that supports only tri-linear filtering is "on-the-fly" filtering [S5, unofficial]. This method leverages the embedded bi-linear texture filtering ability to reduce the number of explicit texture fetches required to compute the bi-cubic values.

1.  **Bi-cubic Optimization**: To calculate a bi-cubic interpolated value, only four bi-linear texture samplings are required. This is achieved by arranging the texture coordinates such that the hardware's internal bi-linear interpolation performs part of the cubic weighting [S5, unofficial].
2.  **Tri-cubic Assembly**: A single tri-cubic sampling is performed by making 8 basic tri-linear samplings. These 8 samples correspond to the 8 corners of the cube surrounding the query point in the "coarse" grid, but each corner's value is itself a bi-cubic interpolation derived from the 8 surrounding voxels. Effectively, this calculates the value determined by 64 nearest voxels [S5, unofficial].

### Pseudocode: Software Tri-Cubic Interpolation
The following pseudocode implements standard tri-cubic interpolation using the Catmull-Rom kernel. It assumes a volume array `V` and handles boundary conditions by clamping or wrapping (implementation detail [G]).

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_kernel(u):
    """
    Catmull-Rom cubic kernel.
    u is the absolute distance from the grid point.
    """
    if u < 0:
        u = -u
    if u < 1:
        return 2.0 * u**3 - 3.0 * u**2 + 1.0
    elif u < 2:
        return -2.0 * u**3 + 3.0 * u**2
    else:
        return 0.0

def tri_cubic_interpolate(volume, x, y, z, width, height, depth):
    """
    Interpolate value at (x, y, z) in volume.
    volume: 3D array [depth][height][width]
    x, y, z: float coordinates in voxel space
    """
    # Determine the 4x4x4 grid of indices
    # x is between floor(x) and ceil(x)
    x0 = floor(x)
    x1 = x0 + 1
    x2 = x0 + 2
    x3 = x0 + 3
    
    y0 = floor(y)
    y1 = y0 + 1
    y2 = y0 + 2
    y3 = y0 + 3
    
    z0 = floor(z)
    z1 = z0 + 1
    z2 = z0 + 2
    z3 = z0 + 3
    
    # Calculate fractional distances for kernel evaluation
    dx0 = x - x0
    dx1 = 1.0 - dx0 # distance to x1
    dx2 = dx0 - 1.0 # distance to x2
    dx3 = dx0 - 2.0 # distance to x3
    
    dy0 = y - y0
    dy1 = 1.0 - dy0
    dy2 = dy0 - 1.0
    dy3 = dy0 - 2.0
    
    dz0 = z - z0
    dz1 = 1.0 - dz0
    dz2 = dz0 - 1.0
    dz3 = dz0 - 2.0
    
    # Evaluate 1D kernels
    kx0 = cubic_kernel(dx0)
    kx1 = cubic_kernel(dx1)
    kx2 = cubic_kernel(dx2)
    kx3 = cubic_kernel(dx3)
    
    ky0 = cubic_kernel(dy0)
    ky1 = cubic_kernel(dy1)
    ky2 = cubic_kernel(dy2)
    ky3 = cubic_kernel(dy3)
    
    kz0 = cubic_kernel(dz0)
    kz1 = cubic_kernel(dz1)
    kz2 = cubic_kernel(dz2)
    kz3 = cubic_kernel(dz3)
    
    # Sum contributions
    result = 0.0
    
    # Iterate over the 4x4x4 neighborhood
    # Indices must be clamped to volume bounds [G]
    indices_x = [x0, x1, x2, x3]
    indices_y = [y0, y1, y2, y3]
    indices_z = [z0, z1, z2, z3]
    
    weights_x = [kx0, kx1, kx2, kx3]
    weights_y = [ky0, ky1, ky2, ky3]
    weights_z = [kz0, kz1, kz2, kz3]
    
    for i in range(4):
        xi = indices_x[i]
        wx = weights_x[i]
        # Clamp xi
        xi = max(0, min(xi, width - 1))
        
        for j in range(4):
            yj = indices_y[j]
            wy = weights_y[j]
            # Clamp yj
            yj = max(0, min(yj, height - 1))
            
            for k in range(4):
                zk = indices_z[k]
                wz = weights_z[k]
                # Clamp zk
                zk = max(0, min(zk, depth - 1))
                
                voxel_val = volume[zk][yj][xi]
                result += voxel_val * wx * wy * wz
                
    return result
```

### Edge Cases and Pitfalls
- **Boundary Conditions**: The kernel extends 2 units beyond the query point. At volume boundaries, indices may go out of bounds. Common strategies include clamping (repeating the edge value), wrapping (toroidal topology), or zero-padding [G]. Clamping is most common for medical volumes to avoid artifacts from outside the body.
- **Precision**: The summation of 64 terms can lead to floating-point precision issues if not handled carefully, though standard `float32` is usually sufficient for visualization [S3, unofficial].
- **Degenerate Input**: If the volume has zero dimensions, the method is undefined.

## Subtopics

### Quasi-Cubic Sampling
A texture sampling technique that applies a quasi-cubic filter kernel to 3D volume textures to reduce aliasing and improve interpolation quality compared to standard linear/trilinear filtering [S4, unofficial]. It approximates cubic behavior using a specific polynomial on the fractional part of the coordinate.
- **Step**: Scale the normalized texture coordinate $u$ by the volume dimension $R$ and add 0.5 to center the texels: $U_{scaled} = u \times R + 0.5$. Extract the fractional part $F = fract(U_{scaled})$ and the integer part. Apply the quasi-cubic kernel to the fractional part: $U_{filtered} = floor(U_{scaled}) + F \times F \times (3.0 - 2.0 \times F)$. Sample the volume texture using the filtered coordinate normalized back to $[0,1]$: $sample = texture(volume, (U_{filtered} - 0.5) / R)$ [S4, unofficial].
- **When to choose**: When hardware supports custom fragment shaders or specific texture filtering modes that allow this polynomial approximation, offering a middle ground between linear and full tri-cubic cost.

### On-the-fly Tri-cubic Filtering
An interpolation method using tri-cubic filtering to improve visualization quality over tri-linear filtering, calculated on-the-fly during ray casting by leveraging embedded bi-linear texture filtering [S5, unofficial].
- **Step**: Calculate bi-cubic interpolated values using only four samplings by leveraging embedded bi-linear texture filtering. Perform a single tri-cubic sampling by making 8 basic tri-linear samplings, effectively calculating the data value determined by 64 nearest voxels [S5, unofficial].
- **Pitfall**: High computational cost. Interactive performance (>20-30 fps) is only guaranteed for small datasets ($\le 256^3$ voxels) or simple techniques (DVR, opaque iso-surface) [S5, unofficial].
- **When to choose**: When running on hardware with efficient tri-linear texture units but no native tri-cubic support, and dataset sizes are manageable.

### Catmull-Rom Interpolation
A second-order interpolation filter that yields a function with a continuous first derivative, reducing banding artifacts compared to linear filters [S6, unofficial]. It is the specific instance of cubic convolution with parameter $a = -0.5$ [S6, unofficial].
- **Formula**: The 1D filter function $h(u)$ is defined piecewise: $h(u) = 2u^3 - 3u^2 + 1$ for $0 \le u < 1$; $h(u) = -2u^3 + 3u^2$ for $1 \le u \le 2$; $h(u) = 0$ for $u > 2$ [S6, unofficial].
- **Step**: Compute the 3D filter value using separability: $h(u,v,w) = h(u)h(v)h(w)$. Implement interpolation as a 3-stage nested loop corresponding to the three separable dimensions [S6, unofficial].
- **Comparison**: Higher filter quality (like cubic) comes at the price of wider spatial extent and larger computational effort compared to linear filters [S6, unofficial]. It interpolates a discrete function with the lowest third-order error among cubic splines [S6, unofficial].

## Operating characteristics
- **Time Complexity**: Considerably high computational costs. In 3D, it requires sampling 64 voxels, compared to 8 for tri-linear [S1, S2, S3, unofficial]. It requires two more multiplications per dimension compared to linear interpolation, leading to a considerable amount of instructions [S3, unofficial].
- **Memory Complexity**: No additional memory storage is required for the kernel itself (it is computed on the fly), but the cache footprint is larger due to the 64-sample access pattern, which may cause cache misses in software implementations [S3, unofficial].
- **Measured Performance**: Interactive performance (>20-30 fps) is only guaranteed for small datasets ($\le 256^3$ voxels) or simple rendering techniques when using on-the-fly filtering [S5, unofficial]. For larger datasets, pre-filtering or lower-order interpolation is typically required [S3, unofficial].
- **Strengths**: Produces significantly smoother results than linear interpolation, reducing the "blocky" appearance and high-frequency leakage artifacts [S1, S2, unofficial]. Allows for smoothing or sharpening via parameter adjustment (in the general cubic convolution family) [S3, unofficial].
- **Limitations**: High computational cost makes it unsuitable for real-time rendering of large volumes without optimization or hardware support [S3, S5, unofficial]. The wider spatial extent can cause blurring of fine details compared to sharper filters like the sinc filter (though sinc has ringing) [S6, unofficial].
- **When to use**: When visual quality is prioritized over performance, or when rendering small-to-medium volumes where smoothness is critical [S3, unofficial].
- **When not to use**: For large-scale medical datasets requiring real-time interaction, or when hardware does not support efficient tri-linear sampling tricks [S5, unofficial].

## Background
Cubic interpolation in volume rendering originates from the broader field of signal processing and image reconstruction, where higher-order polynomials are used to reconstruct continuous signals from discrete samples. The specific application to volume rendering was explored in the early 2000s as hardware capabilities improved. The Catmull-Rom spline, widely used as the default cubic filter, is a classic cardinal spline from computer graphics [S6, unofficial]. The "on-the-fly" tri-cubic optimization was developed to leverage existing GPU texture hardware (bi-linear filtering) to approximate tri-cubic results without dedicated hardware support, as described in implementation aspects of GPU-based volume rendering [S5, unofficial]. The trade-offs between quality and cost were systematically analyzed in various overviews of volume rendering techniques [S1, S3, S6, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S3] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S4] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S5] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
[S6] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
