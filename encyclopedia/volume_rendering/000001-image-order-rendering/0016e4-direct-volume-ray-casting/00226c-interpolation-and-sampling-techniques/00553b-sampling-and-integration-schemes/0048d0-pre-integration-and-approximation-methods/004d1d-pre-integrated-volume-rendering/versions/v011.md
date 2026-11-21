# Pre-integrated Volume Rendering

Also called: PIVR, Pre-integration, Precomputed volume rendering integral, Preintegration Volume Rendering, Pré-integração, Engel, Kraus & Ertl 2001, Pre-integrated transfer function

## Overview
Pre-integrated Volume Rendering (PIVR) accelerates direct volume rendering by pre-computing the volume rendering integral for a ray segment and storing the results in a lookup table for fast retrieval [S1, S4, S8, unofficial]. It solves the problem of aliasing artifacts caused by high-frequency, non-linear transfer functions when sampling sparse ray segments [S1, unofficial][S5]. Unlike standard classification which applies the transfer function to individual sample points, PIVR decouples the integration of the transfer function contribution from the sampling process, allowing the sampling to focus only on volume reconstruction while the pre-computed table handles the optical properties [S8, unofficial]. It sets itself apart from [Pre-integration of Transfer Function Contribution](12225-pre-integration-of-transfer-function-contribution.md) by specifically addressing the scalar field interpolation between segment endpoints to compute accurate color and opacity contributions [S2, S8, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scalar volume data $V$, transfer functions for extinction density $\tau(s)$ and color density $\tilde{c}(s)$, ray segment length $d$ (assumed constant per frame), and scalar values at segment endpoints $s_b$ (back) and $s_f$ (front) [S4, S8, unofficial].
*   **Outputs:** Accumulated color $\tilde{C}$ and opacity $\alpha$ for the segment, which are composited into the final image [S8, unofficial].

**Data Structures:**
1.  **Integral Table:** A 1D lookup table of size 256 entries (for 8-bit scalar data) [S4, unofficial]. Each entry stores four 32-bit floating-point numbers: accumulated color components $(\tilde{R}_A, \tilde{G}_A, \tilde{B}_A)$ and accumulated opacity $T_A$ [S4, unofficial].
    *   Layout: Contiguous memory, `[R, G, B, A]` per entry [S4, unofficial].
    *   Values: $T(s) = \int_0^s \tau(s) ds$ and $K(s) = \int_0^s \tilde{c}(s) ds$ are pre-computed and stored [S4, unofficial].
2.  **Exponential Approximation Parameters:** Four linear segments defined by slopes $m_i$ and intercepts $b_i$ ($i=0..3$) to approximate $1 - \exp(-x)$ [S4, unofficial].

**Numbered Steps:**

1.  **Pre-computation (Offline/Setup):**
    *   Compute the integral functions $T(s)$ and $K(s)$ for all possible scalar values $s \in [0, 255]$ using the user-supplied transfer functions $\tau(s)$ and $\tilde{c}(s)$ [S4, unofficial].
    *   Store these values in the 256-entry 1D table [S4, unofficial].
    *   Compute the linear approximation parameters $m_i, b_i$ for $1 - \exp(-x)$ using a brute-force minimization method for $x \ge 0$ [S4, unofficial].

2.  **Ray Casting Loop (Per Ray):**
    *   Iterate through the volume along the ray $r(t) = \mathbf{o} + t\mathbf{d}$ [S8, unofficial].
    *   For each segment of length $d$, sample the scalar field at the start ($s_{prev}$) and end ($s_{curr}$) of the segment [S4, unofficial].
    *   **Thresholding Check:** If $s_{max} = \max(s_{prev}, s_{curr})$ is below a user-supplied threshold (assuming clamped transfer functions), skip the segment [S4, unofficial].

3.  **Segment Evaluation:**
    *   Determine $s_{max} = \max(s_{prev}, s_{curr})$ and $s_{min} = \min(s_{prev}, s_{curr})$ [S4, unofficial].
    *   Compute the difference $diff = s_{max} - s_{min}$ [S4, unofficial].
    *   Compute the reciprocal $invDiff = 1.0 / diff$ [S4, unofficial].
    *   **Table Lookup:** Access the integral table at indices $s_{max}$ and $s_{min}$ to retrieve $T(s_{max}), K(s_{max})$ and $T(s_{min}), K(s_{min})$ [S4, unofficial].
    *   **Color Computation:**
        $$ \Delta K = K(s_{max}) - K(s_{min}) $$
        $$ \tilde{C} = d \cdot invDiff \cdot \Delta K $$
        This yields the positive raypoint light emission [S4, unofficial].
    *   **Opacity Computation:**
        $$ x = invDiff \cdot (T(s_{max}) - T(s_{min})) $$
        Note that $x \ge 0$ [S4, unofficial].
        Approximate the opacity using the linear segments:
        $$ y_i = m_i \cdot x + b_i \quad \text{for } i \in \{0, 1, 2, 3\} $$
        $$ \alpha = \min(y_0, y_1, y_2, y_3) $$
        This approximates $1 - \exp(-x)$ [S4, unofficial].

4.  **Compositing:**
    *   Combine the computed $\tilde{C}$ and $\alpha$ with the accumulated color and opacity from previous segments using front-to-back compositing [S8, unofficial].

**Formulas:**
The approximate opacity $\alpha$ and emitted light intensity $\tilde{C}$ for an interval between two raypoints with distance $d$ and scalar values $s_b$ and $s_f$ are given by [S4, unofficial]:
$$ \alpha(s_f, s_b, d) \approx 1 - \exp\left(\frac{d}{s_b - s_f} \cdot (T(s_f) - T(s_b))\right) $$
$$ \tilde{C}(s_f, s_b, d) \approx \frac{d}{s_b - s_f} \cdot (K(s_b) - K(s_f)) $$
where $T(s) = \int_0^s \tau(s) ds$ and $K(s) = \int_0^s \tilde{c}(s) ds$. Here $\tau(s)$ is the extinction density and $\tilde{c}(s)$ is the color density [S4, unofficial].

The general volume rendering integral for opacity is [S8, unofficial]:
$$ \alpha = 1 - \exp \left( - \int_{l_i}^{l_o} \lambda(V(R(l))) \times dl \right) $$
where $\lambda$ is the extinction coefficient function, $V$ is the volume field, $R(l)$ is the ray position, and $l_i, l_o$ are the start and end points of the segment [S8, unofficial].

**Parameters:**
*   **Raypoint distance $d$:** Assumed constant for all rays during a frame. Typical values tested: 0.45 and 0.9 grid units [S4, unofficial].
*   **Linear Approximation Parameters ($m_i, b_i$):** Computed once via brute-force method. Specific numerical values are not provided in the sources, only that they minimize error for $x \ge 0$ [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Simplified Alpha Loss:** Using a simplified linear approximation for opacity (e.g., $\alpha \approx \frac{d}{s_b - s_f} (T(s_b) - T(s_f))$) instead of the exponential approximation results in a visible loss in contrast [S4, unofficial].
*   **Thresholding:** Raypoints below a threshold should not be simply discarded if the previous point was above the threshold, as linear interpolation might exceed the threshold within the interval. However, if $s_{max}$ is below the threshold, the entire interval can be safely skipped (assuming clamped transfer functions) [S4, unofficial].
*   **Discretization Artifacts:** The method assumes a linear progression of scalar values between samples, which can introduce artifacts if the scalar field varies non-linearly [S6, unofficial].
*   **High-Dimensional Transfer Functions:** Pre-integration is not applicable without restrictions to transfer functions of dimension greater than one due to the large memory and processing time required to build tables with at least four dimensions (for 2D TFs) [S8, unofficial].

## Subtopics

### [Adaptive Pre-integration](8075-adaptive-pre-integration.md)
Adaptive Pre-integration is a technique applied to texture slicing that pre-computes optimal step sizes for each ray based on pre-integrated emission, opacity, and second-order gradients to guarantee error bounds on ray integration. It is an adaptive approach that overcomes pre-integration artifacts by using an adaptive sampling rate based on the actual information stored in the volume and the transfer function used. It builds on [Pre-integrated Volume Rendering](5327-pre-integrated-volume-rendering.md) by dynamically adjusting the discretization of the pre-integration integral rather than using a fixed slab thickness. Choose it when scalar fields vary highly non-linearly and fixed-step pre-integration artifacts are unacceptable.

### [Direct Volume Rendering with Pre-integration](8968-direct-volume-rendering-with-pre-integration.md)
Direct Volume Rendering with Pre-integration renders adapted tetrahedral meshes using direct volume ray casting, accelerated by pre-integration computed in shaders. It extends pre-integrated volume rendering, which allows for faster evaluation by pre-computing all possible results from the integral and storing them in a table for lookup [Parent], to unstructured volumetric data (tetrahedral meshes). Unlike methods limited to regular grids, this approach handles the geometric complexity of unstructured grids by storing the mesh data structure in textures and using barycentric interpolation for value estimation. Choose it when rendering unstructured tetrahedral meshes requires the speedup of pre-integration.

## Operating characteristics
*   **Time Complexity:** The per-raypoint computation is dominated by SIMD arithmetic operations (MULPS, ADDPS, SUBPS, MINPS, RCPPS) with latencies of 4-6 cycles [S4, unofficial]. The pre-computation of the lookup table is time-consuming, especially for high-dimensional transfer functions [S5].
*   **Memory Complexity:** The 1D table implementation uses 4 Kbytes (256 entries $\times$ 4 floats $\times$ 4 bytes), which fits in the L1 cache, avoiding cache misses associated with larger 2D tables (e.g., 512 Kbytes) [S4, unofficial].
*   **Measured Performance:** On a Pentium-4 2GHz, pre-integrated rendering (d=0.45, simplified alpha) took 0.158s for a 256x256 viewport, compared to 0.295s for standard classification with empty-space skipping [S4, unofficial]. Pre-integration increases speed by a factor of 10 for the linear model compared to baseline [S3, abstract only, unofficial].
*   **Strengths:** Reduces aliasing artifacts caused by high frequencies in non-linear transfer functions [S4, unofficial][S5]. Allows for larger ray step sizes (coarser sampling) while maintaining image quality [S4, unofficial]. Decouples sampling from transfer function evaluation [S8, unofficial].
*   **Limitations:** Assumes linear scalar progression between samples, which can cause artifacts [S6, unofficial]. Not directly applicable to high-dimensional transfer functions without restrictions [S8, unofficial]. The simplified alpha approximation may reduce contrast [S4, unofficial].
*   **When to use:** Suitable for interactive volume rendering on CPUs or GPUs where transfer functions are non-linear and high-frequency, and where memory bandwidth is a bottleneck [S4, S7, unofficial].
*   **When not to use:** When the scalar field varies highly non-linearly within segments and adaptive sampling is not used [S6, unofficial]. When rendering requires high-dimensional transfer functions without applying restrictions, as pre-integration is not applicable to TFs of dimension greater than one without significant memory and processing overhead [S8, unofficial].

## Background
Pre-integrated volume rendering was introduced by Engel et al. in 2001 [S1, S7, unofficial]. It was developed to overcome the problem of high Nyquist frequencies resulting from non-linear transfer functions in texture-based volume rendering [S1, unofficial]. The method splits the numerical integration into two parts: one for the continuous scalar field and one for the transfer function [S1, unofficial]. It was later combined with volumetric clipping and advanced lighting effects by Röttger et al. [S1, unofficial]. The technique has been implemented in software systems like UltraVis [S4, unofficial] and adapted for unstructured grids [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Common Artifacts in Volume Rendering, https://doi.org/10.48550/arxiv.2109.13704, fetched 2026-10-26 (unofficial)
[S3] Real Time Rendering of Deformable and Semi-Transparent Objects by Volume Rendering, https://openalex.org/W2286863887, fetched 2026-10-26 (abstract only, unofficial)
[S4] Using pre-integrated Transfer Functions in an Interactive Software System for Volume, https://doi.org/10.2312/egs.20021047, fetched 2026-10-26 (unofficial)
[S5] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S6] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S7] High-Quality Pre-Integrated Volume Rendering Using ... - ResearchGate, https://www.researchgate.net/publication/2878658_High-Quality_Pre-Integrated_Volume_Rendering_Using_Hardware-Accelerated_Pixel_Shading, fetched 2026-11-03 (unofficial)
[S8] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
