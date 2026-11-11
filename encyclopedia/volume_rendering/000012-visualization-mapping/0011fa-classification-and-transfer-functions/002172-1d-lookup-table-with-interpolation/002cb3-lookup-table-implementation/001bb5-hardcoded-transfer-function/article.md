# Hardcoded Transfer Function

Also called: 1D Lookup Table with Interpolation, Fixed RGBA Array, Static 1D Lookup Table, 1D Transfer Function, 1D data-based TF, 1dTF

## Overview
The Hardcoded Transfer Function (1dTF) maps a single scalar data value directly to a color and opacity tuple $(R, G, B, \alpha)$ using a static lookup table [S2, S4]. It solves the problem of visualizing scalar volume data by assigning appearance based solely on intensity, ignoring spatial context or gradient information [S4]. Unlike [2D Transfer Function Volume Rendering](1295-2d-transfer-function-volume-rendering.md) or [Tent Alpha Transfer Function](3031-tent-alpha-transfer-function.md), it cannot distinguish between features with identical scalar values but different structural contexts, such as boundaries or textures [S2, S4]. It is distinguished by its minimal computational cost and lack of user interaction requirements compared to other transfer function types [S2].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar data value $v$ (e.g., voxel intensity) and a domain range $[x_{min}, x_{max}]$ [S2][S5, unofficial].
*   **Output:** A 4-component vector $(R, G, B, \alpha)$ where each component is in $[0, 1]$ [S3, unofficial][S4].

**Data Structures**
The transfer function is stored as a contiguous array of 4-component vectors [S2][S5, unofficial].
*   **Format:** `float4` (R, G, B, A), where each component is a 32-bit float [S5, unofficial].
*   **Size:** Commonly 256 entries for 8-bit data [S1, unofficial] or 4096 entries for higher precision [S3, unofficial]. The width must be a power of two [S1, unofficial].
*   **Storage:** Uploaded to GPU memory as a 1D CUDA array bound to a texture object (for hardware interpolation) and/or a raw device pointer (for direct lookup) [S5, unofficial].
*   **Class Members (CUDA Implementation):**
    *   `m_count` (int): Number of entries in the table [S5, unofficial].
    *   `m_domain` (float2): Min and max scalar values ($x_{min}, x_{max}$) [S5, unofficial].
    *   `m_device_table` (float4*): Raw device memory for direct lookup [S5, unofficial].
    *   `m_array` (cudaArray_t): 1D CUDA array for texture binding [S5, unofficial].
    *   `m_tex` (cudaTextureObject_t): Texture object for hardware-sampled lookup [S5, unofficial].

**Numbered Steps**

1.  **Initialization (Host to Device):**
    a. Allocate device memory `m_device_table` of size `count * sizeof(float4)` [S5, unofficial].
    b. Copy the host-side RGBA table to `m_device_table` using `cudaMemcpy` with `cudaMemcpyHostToDevice` [S5, unofficial].
    c. Create a channel format descriptor `ch` for `float4` [S5, unofficial].
    d. Allocate a 1D CUDA array `m_array` with width `count`, height 1, using `ch` [S5, unofficial].
    e. Copy the host table to `m_array` using `cudaMemcpy2DToArray` with pitch `count * sizeof(float4)` and height 1 [S5, unofficial].
    f. Create a texture object `m_tex` from `m_array` [S5, unofficial].
        i. Set `res.resType = cudaResourceTypeArray` and `res.res.array.array = m_array` [S5, unofficial].
        ii. Set `td.normalizedCoords = 1` (enable normalized coordinates) [S5, unofficial].
        iii. Set `td.filterMode = cudaFilterModeLinear` (enable linear interpolation) [S5, unofficial].
        iv. Set `td.addressMode[0] = cudaAddressModeClamp` (clamp out-of-bounds access) [S5, unofficial].
        v. Set `td.readMode = cudaReadModeElementType` [S5, unofficial].
        vi. Call `cudaCreateTextureObject` with these descriptors [S5, unofficial].

2.  **Domain Normalization:**
    Map the input scalar value $v$ to a normalized coordinate $x \in [0, 1]$ [S2][S3, unofficial][S4][S5, unofficial]:
    $$ x = \frac{v - x_{min}}{x_{max} - x_{min}} $$
    where $x_{min}$ is `m_domain.x` and $x_{max}$ is `m_domain.y` [S5, unofficial].

3.  **Lookup:**
    Retrieve the RGBA value using one of two methods:
    *   **Hardware Interpolation (Texture Sampling):**
        Compute `normalized_x = x`. Call `tex1D<float4>(m_tex, normalized_x)`. The hardware sampler automatically computes the index and performs linear interpolation between adjacent entries based on the texture settings defined in Step 1f [S5, unofficial].
    *   **Direct Lookup (Nearest Neighbor):**
        Compute the integer index:
        $$ idx = \text{clamp}(\lfloor x \cdot (N-1) \rfloor, 0, N-1) $$
        where $N$ is `m_count` [S3, S5, S8, unofficial]. Return `m_device_table[idx]` [S5, unofficial].

4.  **Opacity Correction (Optional/Contextual):**
    If the rendering pipeline requires adjustment for non-unit sample spacing, opacities may be automatically corrected in software prior to or during lookup [S3, unofficial]. If gradient modulation is enabled (e.g., `Gmom=1`), the alpha value from the LUT is multiplied by a gradient lookup value: $\alpha = \text{GmLUT}[\text{Gradient}] \cdot \alpha_{\text{LUT}}$ [S3, unofficial]. If `Gmom=0`, $\alpha = \alpha_{\text{LUT}}$ [S3, unofficial].

**Parameters**
*   **Table Size ($N$):** 256 or 4096. Must be a power of two [S1, S3, unofficial].
*   **Domain ($x_{min}, x_{max}$):** Defined by the data range or user windowing [S2][S5, unofficial].
*   **Gradient Modulation (`Gmom`):** Boolean flag (0 or 1) determining if gradient magnitude modulates opacity [S3, unofficial].

**Edge Cases and Pitfalls**
*   **Identical Values:** The method cannot differentiate between samples with the same scalar value but different visual contexts (e.g., boundaries vs. homogeneous regions) [S2, S4]. All features with the same grey level are colored identically [S2].
*   **Opacity Distribution:** In hardcoded tables, opacity values are often sparse (e.g., mostly 0.0 or 0.1) and lack explicit scalar-to-index mapping documentation, which can lead to visual artifacts if not carefully designed [S1, unofficial].
*   **Precision:** Output values are typically represented as 8-bit (RGB) or 12-bit (Alpha) repeating-fraction numbers in hardware pipelines [S3, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class HardcodedTransferFunction:
    def __init__(self, table: list, domain_min: float, domain_max: float):
        """
        Initialize the transfer function.
        :param table: List of (r, g, b, a) tuples, length N (power of 2).
        :param domain_min: x_min for normalization.
        :param domain_max: x_max for normalization.
        """
        self.table = table
        self.N = len(table)
        self.domain_min = domain_min
        self.domain_max = domain_max
        self.domain_width = domain_max - domain_min
        
        # Validate power of two
        if self.N & (self.N - 1) != 0:
            raise ValueError("Table size must be a power of two")

    def normalize(self, v: float) -> float:
        """Map scalar value v to [0, 1]."""
        if self.domain_width == 0:
            return 0.0
        x = (v - self.domain_min) / self.domain_width
        # Clamp to [0, 1] for out-of-bounds values
        return max(0.0, min(1.0, x))

    def lookup_hardware(self, v: float) -> tuple:
        """
        Simulate hardware linear interpolation via texture sampling.
        Uses linear interpolation between adjacent entries.
        """
        x = self.normalize(v)
        
        # Calculate float index
        float_idx = x * (self.N - 1)
        idx_low = int(math.floor(float_idx))
        idx_high = min(idx_low + 1, self.N - 1)
        
        # Interpolation factor
        t = float_idx - idx_low
        
        r_low, g_low, b_low, a_low = self.table[idx_low]
        r_high, g_high, b_high, a_high = self.table[idx_high]
        
        r = r_low + t * (r_high - r_low)
        g = g_low + t * (g_high - g_low)
        b = b_low + t * (b_high - b_low)
        a = a_low + t * (a_high - a_low)
        
        return (r, g, b, a)

    def lookup_direct(self, v: float) -> tuple:
        """
        Simulate direct nearest-neighbor lookup.
        """
        x = self.normalize(v)
        idx = int(math.floor(x * (self.N - 1)))
        # Clamp index
        idx = max(0, min(idx, self.N - 1))
        return self.table[idx]

    def apply_gradient_modulation(self, rgba: tuple, gradient: float) -> tuple:
        """
        Optional: Modulate alpha by gradient magnitude if Gmom=1.
        :param rgba: (r, g, b, a) from lookup.
        :param gradient: Gradient magnitude.
        :return: (r, g, b, a * gradient_factor).
        """
        r, g, b, a = rgba
        # Simplified gradient modulation as per VolumePro logic [S3, unofficial]
        # Assuming GmLUT[Gradient] is a lookup or function of gradient
        # For this generic implementation, we assume a direct multiplier or 1.0 if not specified
        # Note: Specific GmLUT implementation is not provided in sources for generic case.
        # We return original alpha if Gmom=0 (default behavior for pure 1dTF)
        return (r, g, b, a)
```

## Subtopics

### [2D Transfer Function Volume Rendering](1295-2d-transfer-function-volume-rendering.md)
Extends the 1D Lookup Table with Interpolation method by mapping both the scalar data value $f$ and the local gradient magnitude $|\nabla f|$ to color and opacity. It solves the problem of overlapping data-value intervals in 1D transfer functions, where different materials or boundaries share similar scalar values but differ in their local structure. By using gradient magnitude as a second dimension, it uniquely isolates material boundaries and homogeneous regions, providing superior tissue differentiation and boundary visualization compared to [Hardcoded Transfer Function](1098-hardcoded-transfer-function.md).

### [Tent Alpha Transfer Function](3031-tent-alpha-transfer-function.md)
A hardcoded 1D RGBA transfer texture generation method for fluorescence microscopy data that maps 8-bit scalar values to color and opacity using a tent-shaped alpha curve and gamma corrections. It provides a parametric opacity profile centered on a user-defined peak value. It builds on the [Hardcoded Transfer Function](1098-hardcoded-transfer-function.md) by defining the color and opacity as a static float array generated from a closed-form mathematical profile rather than a manually edited lookup table.

### Fire Color Palette
A piecewise linear color mapping function that maps a value in $[0, 1]$ to a gradient from gray to dark gray to red to orange to yellow [S7, unofficial].
*   **Steps:**
    1.  Clamp input $d$ to $[0.0, 1.0]$.
    2.  If $x < 0.25$, interpolate between `gray` (0.4, 0.4, 0.4) and `darkgray` (0.2, 0.2, 0.2) using $t = x \cdot 4.0$.
    3.  If $0.25 \le x < 0.5$, interpolate between `darkgray` and `red` (1.0, 0.0, 0.0) using $t = x \cdot 4.0 - 1.0$.
    4.  If $0.5 \le x < 0.75$, interpolate between `red` and `orange` (1.0, 0.6, 0.0) using $t = x \cdot 4.0 - 2.0$.
    5.  If $x \ge 0.75$, interpolate between `orange` and `yellow` (1.7, 1.3, 1.0) using $t = x \cdot 4.0 - 3.0$. Note: Components can exceed 1.0 [S7, unofficial].

### Exponential Color Transfer Function
Maps volume intensity to color and alpha using an exponential function for alpha and linear interpolation for color [S8, unofficial].
*   **Steps:**
    1.  Define high color as white: `vec3 high = vec3(1.0, 1.0, 1.0)`.
    2.  Define low color as black: `vec3 low = vec3(0.0, 0.0, 0.0)`.
    3.  Compute color: `vec4(intensity * high + (1.0 - intensity) * low, alpha)`.
*   **Formula:** $\alpha = \frac{e^{intensity} - 1}{e^1 - 1}$ [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for lookup [S2].
*   **Memory Complexity:** $O(N)$ where $N$ is the table size (e.g., 256 or 4096 entries) [S2][S3, unofficial].
*   **Performance:** Requires the lowest amount of computation power and memory compared to other TF types (e.g., 2D, curvature-based, or texture-based TFs) [S2].
*   **Strengths:** High safety regarding errors that lead to misinterpretation; little user interaction required; simple implementation [S2].
*   **Limitations:** Cannot distinguish features with identical scalar values but different contexts (e.g., boundaries vs. interiors); limited expressiveness for multi-channel data [S2, S4].
*   **When to Use:** Scalar data visualization where intensity is the sole distinguishing feature and performance is critical.
*   **When Not to Use:** When differentiating between materials with similar intensities but different structures (e.g., tissue boundaries) is required; use [2D Transfer Function Volume Rendering](1295-2d-transfer-function-volume-rendering.md) instead.

## Background
The 1D data-based transfer function (1dTF) is a foundational technique in volume rendering, serving as a reference point for more complex transfer functions [S2]. It was widely implemented in early real-time ray-casting systems such as VolumePro, which used a 4096-entry classification LUT loaded prior to rendering [S3, unofficial]. The method is described in surveys of transfer functions suitable for volume rendering, highlighting its trade-offs in computational cost versus feature emphasis [S2]. It is commonly used in medical imaging and scientific visualization for initial data exploration due to its simplicity and low resource usage [S2, S4].

## Sources
[S1] extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, fetched 2026-10-26 (unofficial)
[S2] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S3] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S4] Dimensionality reduction on multi-dimensional transfer functions for ..., https://cseweb.ucsd.edu/~hskim/files/IVS_ivs20106.pdf, fetched 2026-10-27
[S5] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S6] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
[S7] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
[S8] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
