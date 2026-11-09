# Fast tri-cubic interpolation

Also called: Tri-cubic interpolation

## Overview
Fast tri-cubic interpolation is a GPU-accelerated technique for reconstructing volume data using a cubic B-spline filter, designed to reduce the number of texture fetches required for the convolution sum [S1]. It solves the problem of high computational cost associated with evaluating the 64-term tri-cubic convolution sum by leveraging GPU linear filtering capabilities and the separability of tensor-product B-splines [S1]. It sets itself apart from standard tri-cubic interpolation by reducing the fetch count from 64 to 8 via pre-computed weight/offset lookups and multi-channel texture storage [S1][S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume texture (source data), a query position $\mathbf{x} = (x, y, z)$ in voxel space.
- Output: The interpolated scalar value $V(\mathbf{x})$.

**Data Structures:**
1. **Lookup Texture:** A single 1D floating-point RGBA texture [S2, S3, unofficial].
   - Layout: Each channel (R, G, B, A) stores one "tile" of the discretized reconstruction kernel [S2, S3, unofficial].
   - For a kernel of width 4, there are 4 tiles [S2, S3, unofficial]. A single texture fetch at position $t$ returns 4 weight values $(w_0, w_1, w_2, w_3)$ corresponding to the four neighboring texels [S2, S3, unofficial].
   - Format: Floating-point to support precise blending calculations [S2, S3, unofficial].
2. **Weight/Offset Lookup:** Pre-computed functions $g_i(x)$ and $h_i(x)$ stored in a lookup texture to further optimize the 1D pass [S1].

**Method Steps:**
The method exploits the separability of tensor-product B-splines [S1]. A 3D tri-cubic filter with 64 summands is evaluated using only 8 tri-linear texture fetches [S1].

1. **1D Reconstruction (Separable Pass):**
   For a single dimension with coordinate $x$:
   a. Compute the integer part $[x]$ and fractional part [S1].
   b. Retrieve pre-computed weight functions $g_0(x), g_1(x)$ and offset functions $h_0(x), h_1(x)$ from the lookup texture [S1].
      $$ g_0(x) = w_0(x) + w_1(x) $$
      $$ h_0(x) = 1 - \frac{w_1(x)}{w_0(x) + w_1(x)} $$
      $$ g_1(x) = w_2(x) + w_3(x) $$
      $$ h_1(x) = 1 + \frac{w_3(x)}{w_2(x) + w_3(x)} $$
      where $w_k(x)$ are the cubic B-spline weights [S1].
   c. Perform two linear texture fetches using hardware linear filtering:
      $$ f_{\text{left}} = \text{texel}([x] - h_0(x)) $$
      $$ f_{\text{right}} = \text{texel}([x] + h_1(x)) $$
   d. Compute the 1D result:
      $$ f(x) = g_0(x) f_{\text{left}} + g_1(x) f_{\text{right}} $$
   *Note: This step reduces 4 fetches to 2 fetches + 1 weighted sum [S1].*

2. **3D Extension:**
   Apply the 1D reconstruction separably along X, Y, and Z axes [S1]. Due to separability, the total 64-term sum is computed using $2 \times 2 \times 2 = 8$ tri-linear texture fetches [S1].

3. **Hardware Implementation (Multi-Pass Strategy):**
   Because current hardware does not support blending with full floating-point precision in a single pass for the full kernel width, the filtering is divided into four passes [S2, S3, unofficial].
   a. **Pass 1-4:** Each pass computes the contribution of 16 neighboring samples (4 neighbors $\times$ 4 channels/tiles) [S3, unofficial]. This is achieved by storing the kernel tiles in a single 1D RGBA texture, where one fetch returns 4 weights [S2, S3, unofficial].
   b. **Blending Pass:** A separate pass sums the four intermediate sub-results together [S3, unofficial]. Direct blending is avoided due to precision limitations [S2, S3, unofficial].
   c. **Storage:** The filtered slice is stored in a texture for subsequent processing (e.g., inverse FFT in frequency domain rendering) [S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fetch_1d(texture, coord):
    """Returns a 4-component vector from a 1D texture at coord."""
    pass

def fetch_3d(texture, coord):
    """Returns a scalar from a 3D texture at coord using tri-linear interpolation."""
    pass

def fast_tricubic_1d(x, lookup_tex, volume_tex):
    """
    Performs 1D fast tri-cubic interpolation using pre-computed g/h lookups.
    """
    # 1. Retrieve pre-computed weight and offset functions
    g0, g1, h0, h1 = fetch_1d(lookup_tex, x)
    
    # 2. Compute effective sample coordinates
    int_x = math.floor(x)
    coord_left = int_x - h0
    coord_right = int_x + h1
    
    # 3. Fetch values using hardware linear interpolation
    val_left = fetch_3d(volume_tex, (coord_left, 0.0, 0.0))
    val_right = fetch_3d(volume_tex, (coord_right, 0.0, 0.0))
    
    # 4. Compute weighted sum
    return g0 * val_left + g1 * val_right

def fast_tricubic_3d(x, y, z, lookup_tex, volume_tex):
    """
    3D tri-cubic interpolation using separability.
    Requires 8 tri-linear fetches.
    """
    # Compute effective coordinates for each axis
    int_x, int_y, int_z = math.floor(x), math.floor(y), math.floor(z)
    
    g0x, g1x, h0x, h1x = fetch_1d(lookup_tex, x)
    g0y, g1y, h0y, h1y = fetch_1d(lookup_tex, y)
    g0z, g1z, h0z, h1z = fetch_1d(lookup_tex, z)
    
    x_l, x_r = int_x - h0x, int_x + h1x
    y_l, y_r = int_y - h0y, int_y + h1y
    z_l, z_r = int_z - h0z, int_z + h1z
    
    # Fetch 8 tri-linear samples at the effective coordinates
    v000 = fetch_3d(volume_tex, (x_l, y_l, z_l))
    v001 = fetch_3d(volume_tex, (x_l, y_l, z_r))
    v010 = fetch_3d(volume_tex, (x_l, y_r, z_l))
    v011 = fetch_3d(volume_tex, (x_l, y_r, z_r))
    v100 = fetch_3d(volume_tex, (x_r, y_l, z_l))
    v101 = fetch_3d(volume_tex, (x_r, y_l, z_r))
    v110 = fetch_3d(volume_tex, (x_r, y_r, z_l))
    v111 = fetch_3d(volume_tex, (x_r, y_r, z_r))
    
    # Combine along X
    v00 = g0x * v000 + g1x * v100
    v01 = g0x * v001 + g1x * v101
    v10 = g0x * v010 + g1x * v110
    v11 = g0x * v011 + g1x * v111
    
    # Combine along Y
    v0 = g0y * v00 + g1y * v01
    v1 = g0y * v10 + g1y * v11
    
    # Combine along Z
    return g0z * v0 + g1z * v1
```

**Parameters:**
- Kernel Width: 4 samples [S3, unofficial].
- Texture Format: 1D floating-point RGBA [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Hardware Precision:** Hardware does not support blending with floating-point precision in intermediate passes. A separate blending pass is required to sum the 4 sub-results [S2, S3, unofficial].
- **Fragment Program Length:** ATI GPUs have fragment program length limits, requiring the 4-pass approach. NVIDIA GPUs support longer programs, allowing the passes to be folded into a single pass [S2, S3, unofficial].
- **Periodicity:** The weights are periodic in the sample positions of the input texture [S1].

## Subtopics

### [Adaptive Filter Size for Rendering Time Reduction](861-adaptive-filter-size-for-rendering-time-reduction.md)
A technique to reduce rendering time by adaptively selecting smaller interpolation filters for low-energy frequency components. It builds on Fast tri-cubic interpolation by optimizing the convolution cost during slice extraction in frequency-domain rendering.

### [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md)
A parameterized family of reconstruction filters. It sets itself apart by defining the specific mathematical kernel and parameter space ($B, C$) rather than the hardware-accelerated fetch reduction technique used in Fast tri-cubic interpolation.

## Operating characteristics
- **Time Complexity:** Reduces the number of texture fetches from 64 (naive tri-cubic) to 8 (separable + g/h optimization) [S1]. This increases resampling performance [S2, S3, unofficial].
- **Memory Complexity:** Requires additional memory for the 1D lookup texture storing $g_i$ and $h_i$ functions, and the 1D RGBA texture storing kernel tiles [S1][S2, S3, unofficial].
- **Performance:** Measured on ATI and NVIDIA GPUs. ATI requires multi-pass due to fragment program limits, while NVIDIA can fold passes into one [S2, S3, unofficial]. The method is specifically effective in frequency-domain volume rendering pipelines where resampling is a bottleneck [S2, S3, unofficial].
- **Strengths:** High-quality reconstruction (cubic B-spline), reduced fetch count, hardware-friendly.
- **Limitations:** Requires floating-point texture support (historically limited), multi-pass overhead on some hardware, precision loss in intermediate blending if not handled separately [S2, S3, unofficial].

## Background
Fast tri-cubic interpolation was introduced in the context of real-time ray-casting and advanced shading of discrete isosurfaces, and later applied to frequency-domain volume rendering [S1][S2, S3, unofficial]. The core technique of reducing texture fetches using linear filtering and pre-computed weights was detailed by Weiler et al. in 2000 [S1]. It was further adapted for GPU-based frequency domain rendering by Viola et al. in 2004, who implemented the multi-pass strategy and kernel tile storage [S2, S3, unofficial].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S3] Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S4] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
