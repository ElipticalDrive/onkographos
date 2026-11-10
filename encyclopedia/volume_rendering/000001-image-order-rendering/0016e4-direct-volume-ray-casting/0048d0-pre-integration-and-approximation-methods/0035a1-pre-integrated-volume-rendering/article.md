# Pre-integrated Volume Rendering

Also called: PIVR, Pre-integration (Pré-integração), Pré-integração

## Overview
Pre-integrated Volume Rendering (PIVR) is a technique that accelerates direct volume rendering by pre-computing the integration of transfer functions over a ray interval and storing the results in a lookup table for on-the-fly evaluation [S1, unofficial][S2, unofficial]. It solves the problem of aliasing artifacts caused by high Nyquist frequencies in non-linear transfer functions by decoupling the integration of the transfer function contribution from the sampling process, allowing the sampling to focus solely on volume reconstruction [S6, unofficial]. Unlike standard point-sampling methods, PIVR renders the volume segment by segment rather than point by point, using the scalar values at the endpoints of a segment and the segment length to compute color and opacity contributions [S3][S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A scalar volume dataset $V$, a transfer function defining extinction density $\tau(s)$ and color density $\tilde{c}(s)$, and a ray path through the volume.
*   **Outputs:** A 2D image with accumulated color and opacity.

**Data Structures:**
*   **1D Integral Table:** A table of 256 entries (for 8-bit scalar data). Each entry contains four 32-bit floating-point numbers: accumulated color components $(\tilde{R}_A, \tilde{G}_A, \tilde{B}_A)$ and accumulated opacity $T_A$. Total size: 4 Kbytes. The layout is contiguous in memory: $[R, G, B, A]$ per entry [S2, unofficial].
*   **Scalar Field:** Sampled values along the viewing ray, defining a one-dimensional piecewise linear scalar field [S1, unofficial].

**Pre-computation:**
The integral functions $T(s)$ and $K(s)$ are computed from the user-supplied transfer functions $\tau(s)$ and $\tilde{c}(s)$:
$$ T(s) = \int_0^s \tau(s) ds $$
$$ K(s) = \int_0^s \tilde{c}(s) ds $$
These values are stored in the 1D table. The table must be refreshed whenever the transfer function is modified [S2, unofficial][S1, unofficial].

**Rendering Steps:**
1.  **Sampling:** Sample the scalar field along the viewing ray. This sampling has its own Nyquist frequency, independent of the transfer function [S1, unofficial].
2.  **Segment Definition:** Define segments between consecutive ray points. For linear interpolation, a ray defines a 1D portion of the volume consisting of segments where the sampled field value varies linearly [S6, unofficial].
3.  **Endpoint Retrieval:** For each segment, obtain the current scalar value $s_{current}$ and the previous scalar value $s_{prev}$. Determine $s_{max} = \max(s_{current}, s_{prev})$ and $s_{min} = \min(s_{current}, s_{prev})$ [S2, unofficial].
4.  **Table Lookup:** Perform two sequential table accesses using indices derived from $s_{max}$ and $s_{min}$ (assuming 8-bit quantization, indices are $s_{max}$ and $s_{min}$). Retrieve $T(s_{max}), K(s_{max})$ and $T(s_{min}), K(s_{min})$ [S2, unofficial].
5.  **Color Computation:** Compute the emitted light intensity $\tilde{C}$ for the segment:
    $$ \tilde{C}(s_f, s_b, d) \approx \frac{d}{s_b - s_f} \cdot (K(s_b) - K(s_f)) $$
    where $s_b = s_{max}$, $s_f = s_{min}$, and $d$ is the distance between ray points [S2, unofficial][S6, unofficial].
6.  **Opacity Computation:** Compute the opacity $\alpha$ for the segment:
    $$ \alpha(s_f, s_b, d) \approx 1 - \exp\left(\frac{d}{s_b - s_f} \cdot (T(s_f) - T(s_b))\right) $$
    This involves computing $x = \frac{T(s_{max}) - T(s_{min})}{s_{max} - s_{min}}$ and approximating $1 - \exp(-x)$ [S2, unofficial].
7.  **Compositing:** Compose the color and opacity contributions of all segments in order (front-to-back) to generate the final image [S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preintegrated_volume_rendering(volume, transfer_function, ray_length_d):
    """
    volume: 3D array of scalar values
    transfer_function: function mapping scalar s to (tau(s), c(s))
    ray_length_d: constant distance between ray points
    """
    # Pre-computation
    # T[s] = integral_0^s tau(u) du
    # K[s] = integral_0^s c(u) du
    # Table size 256 entries, each entry is [R, G, B, A] floats
    table = compute_integral_table(transfer_function)
    
    # Approximation parameters for 1 - exp(-x)
    # Computed once via brute-force method to minimize error for x >= 0
    # m_i, b_i are linear coefficients for 4 segments
    m = [m0, m1, m2, m3]
    b = [b0, b1, b2, b3]
    
    def approx_opacity(x):
        # x >= 0
        y0 = m[0] * x + b[0]
        y1 = m[1] * x + b[1]
        y2 = m[2] * x + b[2]
        y3 = m[3] * x + b[3]
        return min(y0, y1, y2, y3)

    # Raycasting Loop
    for each ray:
        accumulated_color = (0, 0, 0)
        accumulated_opacity = 0.0
        prev_s = sample_scalar(ray, 0) # First sample
        
        for step in range(1, num_steps):
            curr_s = sample_scalar(ray, step)
            
            # Thresholding/Skipping
            # If max(s_curr, s_prev) < threshold, skip segment
            if max(curr_s, prev_s) < threshold:
                prev_s = curr_s
                continue
                
            s_max = max(curr_s, prev_s)
            s_min = min(curr_s, prev_s)
            
            # Table Lookup
            # Indices are s_max and s_min (assuming 8-bit quantization)
            entry_max = table[s_max]
            entry_min = table[s_min]
            
            # Extract K and T values
            # K is color integral, T is opacity integral
            # Assuming entry is [R, G, B, A] where A is T
            K_max = entry_max[:3] # RGB
            T_max = entry_max[3]
            K_min = entry_min[:3]
            T_min = entry_min[3]
            
            # Color Computation
            # delta_K = K(s_max) - K(s_min)
            delta_K = K_max - K_min
            # C_tilde = d / (s_max - s_min) * delta_K
            if s_max == s_min:
                segment_color = (0, 0, 0)
            else:
                segment_color = (ray_length_d / (s_max - s_min)) * delta_K
            
            # Opacity Computation
            # x = (T(s_max) - T(s_min)) / (s_max - s_min)
            if s_max == s_min:
                x = 0
            else:
                x = (T_max - T_min) / (s_max - s_min)
            
            segment_opacity = approx_opacity(x)
            
            # Front-to-Back Compositing
            # C_out = C_in + (1 - alpha_in) * C_seg
            # alpha_out = alpha_in + (1 - alpha_in) * alpha_seg
            alpha_seg = segment_opacity
            color_seg = segment_color
            
            accumulated_color = accumulated_color + (1 - accumulated_opacity) * color_seg
            accumulated_opacity = accumulated_opacity + (1 - accumulated_opacity) * alpha_seg
            
            prev_s = curr_s
            
        write_pixel(accumulated_color, accumulated_opacity)
```

**Parameters:**
*   **Raypoint distance $d$:** Assumed constant for all rays during a frame. Typical values tested: 0.45 and 0.9 grid units [S2, unofficial].
*   **Linear Approximation Parameters ($m_i, b_i$):** Computed once via brute-force method to minimize error for $x \geq 0$. Specific numerical values are not provided in the sources [S2, unofficial].
*   **Threshold:** User-supplied value to discard noise. Raypoints below threshold can be skipped if $s_{max}$ is below threshold [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Division by zero if $s_{max} = s_{min}$. Handle by setting color to 0 and opacity calculation to $x=0$ [G].
*   **Thresholding:** Raypoints below a threshold should not be simply discarded if the previous point was above the threshold, as linear interpolation might exceed the threshold. However, if $s_{max}$ is below the threshold, the entire interval is below the threshold (assuming clamped transfer functions) and can be safely skipped [S2, unofficial].
*   **Simplified Alpha Loss:** Using a simplified opacity formula (linear approximation instead of exponential) results in a visible loss in contrast [S2, unofficial].
*   **Discretization Artifacts:** Pre-integration assumes linear progression of scalar values between slices, which can introduce artifacts [S4, unofficial].
*   **High-Dimensional TFs:** Pre-integration is not applicable without restrictions to transfer functions of dimension greater than one due to large memory and processing time requirements [S6, unofficial].

## Subtopics

### [Adaptive Pre-integration](8075-adaptive-pre-integration.md)
Adaptive Pre-integration is a technique applied to texture slicing that pre-computes optimal step sizes for each ray based on pre-integrated emission, opacity, and second-order gradients to guarantee error bounds on ray integration. It is an adaptive approach that overcomes pre-integration artifacts by using an adaptive sampling rate based on the actual information stored in the volume and the transfer function used. It builds on [Pre-integrated Volume Rendering](5327-pre-integrated-volume-rendering.md) by dynamically adjusting the discretization of the pre-integration integral rather than using a fixed slab thickness.

### [Direct Volume Rendering with Pre-integration](8968-direct-volume-rendering-with-pre-integration.md)
Direct Volume Rendering with Pre-integration renders adapted tetrahedral meshes using direct volume ray casting, accelerated by pre-integration computed in shaders. It extends pre-integrated volume rendering, which allows for faster evaluation by pre-computing all possible results from the integral and storing them in a table for lookup [Parent], to unstructured volumetric data (tetrahedral meshes). Unlike methods limited to regular grids, this approach handles the geometric complexity of unstructured grids by storing the mesh data structure in textures and using barycentric interpolation for value estimation.

## Operating characteristics
*   **Time Complexity:** Dominated by SIMD arithmetic operations (MULPS, ADDPS, SUBPS, MINPS, RCPPS) with latencies of 4-6 cycles per raypoint [S2, unofficial].
*   **Memory Complexity:** 3 memory reads per raypoint (1 previous value, 2 table entries), totaling 33 bytes. The 4KB 1D table fits in L1 cache, avoiding cache misses associated with larger 2D tables [S2, unofficial].
*   **Measured Performance:** On a Pentium-4 2GHz, pre-integrated rendering (d=0.45, simplified alpha) took 0.158s for a 256x256 image, compared to 0.295s for standard classification (d=0.45) [S2, unofficial]. Performance of 4 fps for a 256x256 viewport was achieved using a 200 MHz CPU and an nVidia GeForce3 [S1, unofficial].
*   **Strengths:** Reduces aliasing artifacts from high frequencies in transfer functions. Faster than 2D table lookup on CPUs due to L1 cache efficiency. High image quality for low-resolution data and non-linear transfer functions [S2, unofficial][S5, unofficial].
*   **Limitations:** Not applicable to high-dimensional transfer functions without restrictions. Discretization assumes linear scalar progression, potentially introducing artifacts. Simplified alpha approximation causes contrast loss [S2, unofficial][S4, unofficial][S6, unofficial].
*   **When to use:** When rendering regular grid volume data with 1D transfer functions on CPU or GPU, especially when aliasing is a concern and performance is critical.
*   **When not to use:** When using high-dimensional transfer functions or when exact opacity integration is required without approximation artifacts.

## Background
Pre-integrated volume rendering was introduced by Engel et al. in 2001 [Engel01]. It improves upon basic texture mapping by overcoming high Nyquist frequencies from non-linear transfer functions. The method separates the DVR integral into two parts: one for the continuous scalar value and one for TF parameters (colors) and extinction coefficient [S3]. It was later combined with volumetric clipping and advanced lighting effects by Röttger et al. [Röttger03]. The technique is implemented in the UltraVis software system [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Using pre-integrated Transfer Functions in an Interactive Software System for Volume, https://doi.org/10.2312/egs.20021047, fetched 2026-10-26 (unofficial)
[S3] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S4] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S5] High-Quality Pre-Integrated Volume Rendering Using ... - ResearchGate, https://www.researchgate.net/publication/2878658_High-Quality_Pre-Integrated_Volume_Rendering_Using_Hardware-Accelerated_Pixel_Shading, fetched 2026-11-03 (unofficial)
[S6] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
