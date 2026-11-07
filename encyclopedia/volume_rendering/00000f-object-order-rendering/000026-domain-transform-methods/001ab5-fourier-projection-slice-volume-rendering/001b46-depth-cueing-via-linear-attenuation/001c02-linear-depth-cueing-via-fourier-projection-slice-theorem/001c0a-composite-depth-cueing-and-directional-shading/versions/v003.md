# Composite Depth Cueing and Directional Shading

Also called: Composite Shading Model, Hemispherical Illumination Shading

## Overview
Composite Depth Cueing and Directional Shading is a frequency-domain volume rendering technique that integrates linear attenuation (depth cueing) and hemispherical directional shading into a single rendering pass [S1, S2, unofficial]. It solves the problem of high computational cost associated with applying both effects in the spatial domain by leveraging the Fourier Projection-Slice Theorem [S1, S2, unofficial]. Unlike standard spatial rendering which requires $O(n^3)$ operations for these effects, this method reduces complexity to $O(n^2)$ [S1, S2, unofficial]. It is distinct from its sibling [Combined Depth Cueing and Shading](1170-combined-depth-cueing-and-shading.md) by specifically addressing the order of operations to optimize performance, often using a "reversed order" approximation to avoid multiple convolutions [S1, S2, unofficial].

## How it works
The method replaces parallel illumination with illumination from an emissive hemisphere to avoid the nonlinear absolute value operator $|N \cdot L|$, allowing the lighting direction to be factored out of the integral for Fourier projection-slice evaluation [S1, unofficial]. It combines linear attenuation and hemispherical illumination, requiring 16 precomputed spectra [S1, unofficial].

**Inputs and Outputs**
*   **Inputs:**
    *   Volume of albedo $\Omega(x,y,z)$ [S1, unofficial].
    *   Orientation field $N(x,y,z)$, typically estimated from scalar input data using a central difference operator [S1, unofficial].
    *   Viewing direction $V$ and Lighting direction $L$ (unit vectors) [S1, S2, unofficial].
    *   Attenuation coefficient $\tau$ [S2, unofficial].
*   **Outputs:**
    *   Rendered image intensity $I$ [S1, S2, unofficial].

**Data Structures**
The algorithm relies on 16 precomputed 3D spectra derived from the volume and orientation fields.
*   **VL Vector:** A 12-element vector of products of viewing direction components $V$ and lighting direction components $L$:
    $$VL = (V_x L_x, V_x L_y, V_x L_z, V_y L_x, V_y L_y, V_y L_z, V_z L_x, V_z L_y, V_z L_z, V_x, V_y, V_z, L_x, L_y, L_z)$$
    *Note: The source lists 13 elements in the formula definition but refers to a 12-element vector in the text description. The formula explicitly lists 13 terms.* [S1, S2, unofficial]
*   **DN Vector:** A 12-element vector of products of displacement direction components $D$ and orientation components $N$:
    $$DN = (D_x N_x, D_x N_y, D_x N_z, D_y N_x, D_y N_y, D_y N_z, D_z N_x, D_z N_y, D_z N_z, D_x, D_y, D_z, N_x, N_y, N_z)$$
    *Note: Similar to VL, the formula lists 13 terms.* [S1, S2, unofficial]

**Steps**
1.  **Precomputation:**
    a.  Create four copies of the volume $\Omega(x,y,z)$.
    b.  Multiply each copy by the appropriate component of the orientation field $N$: one by $1$, one by $N_x$, one by $N_y$, and one by $N_z$ [S1, unofficial].
    c.  Compute the 3D Fourier Transform of these four modified volumes to yield four 3D spectra [S1, unofficial].
    d.  To support the composite model, the integrand is multiplied by linear attenuation and hemispherical illumination terms, resulting in a total of 16 precomputed spectra [S1, unofficial].

2.  **Per-view Rendering:**
    a.  For the current viewing direction $V$ and lighting direction $L$, extract slices from each of the 16 precomputed spectra [S1, unofficial].
    b.  Inverse Fourier Transform each slice to obtain spatial domain images [S1, unofficial].
    c.  Weight each inverse-transformed slice by the corresponding component of the current lighting direction $L$ (and viewing direction $V$ for attenuation) [S1, unofficial].
    d.  Sum the weighted slices to produce the final image [S1, unofficial].

**Formulas**
The intensity $I$ is computed as [S1, S2, unofficial]:

$$I = \tau \frac{\pi}{2} VL \cdot \Pi_V \left[ \left(\frac{1}{2} + \frac{1}{2} DN(x,y,z)\right) I_s \Omega(x,y,z) \right] + \tau \frac{\pi}{2} c_3 \Pi_V [I_s \Omega(x,y,z)]$$

Where:
*   $\Pi_V$ is the Fourier projection operator along viewing direction $V$ [S1, S2, unofficial].
*   $I_s$ is the intensity of the light source [S1, unofficial].
*   $c_3$ is a constant term defined as:
    $$c_3 = \frac{1}{2} - \frac{1}{2} \sum_i VL_i$$
    *Note: The summation index $i$ ranges over the components of the VL vector.* [S1, S2, unofficial]

**Parameters**
*   $\Omega(x,y,z)$: Volume of albedo [S1, unofficial].
*   $c_3$: Constant term dependent on $V$ and $L$ [S2, unofficial].

**Edge Cases and Pitfalls**
*   The composite model ignores emission $\rho C$ from the depth cueing equation [S1, unofficial]. In this context, emission can be interpreted as ambient illumination [S1, unofficial].
*   The high constant factor makes the algorithm slower than conventional rendering for small volumes, despite being asymptotically less expensive [S1, unofficial].

## Subtopics

### [Combined Depth Cueing and Shading](1170-combined-depth-cueing-and-shading.md)
Combined Depth Cueing and Shading is a frequency-domain volume rendering technique that integrates linear attenuation (depth cueing) and hemispherical directional shading into a single rendering pass. It solves the problem of high computational cost associated with applying both effects in the spatial domain by leveraging the Fourier Projection-Slice Theorem. Unlike standard spatial rendering which requires $O(n^3)$ operations for these effects, this method reduces complexity to $O(n^2)$. It is distinct from its sibling [Composite Depth Cueing and Directional Shading](1143-composite-depth-cueing-and-directional-shading.md) by specifically addressing the order of operations to optimize performance, often using a "reversed order" approximation to avoid multiple convolutions.

## Operating characteristics
*   **Time Complexity:** Asymptotically $O(n^2)$, which is less expensive than conventional spatial rendering $O(n^3)$ [S1, S2, unofficial].
*   **Memory Complexity:** Requires 16 precomputed spectra. For a $128^3$ voxel cube implementation, memory usage is 256 megabytes [S1, unofficial].
*   **Measured Performance:** On a 36 MHz Silicon Graphics 4D/320 VGX with a sequential C implementation, rendering time was over a minute per view for the composite model, mostly due to paging [S1, unofficial]. This is significantly slower than the 6 seconds required for simpler models (e.g., linear depth cueing alone) on the same hardware [S1, unofficial].
*   **Strengths:** Avoids the nonlinear absolute value operator $|N \cdot L|$ present in parallel illumination models, allowing efficient Fourier evaluation [S1, unofficial]. The resulting shading is less sensitive to surface orientation, similar to perception on a cloudy day, and avoids negative light artifacts from backward-facing normals [S1, unofficial].
*   **Limitations:** High constant factor makes it slower than conventional rendering for small volumes [S1, unofficial]. Ignores emission/ambient illumination [S1, unofficial].

## Background
The method was developed by Marc Levoy as part of his work on volume rendering using the Fourier Projection-Slice Theorem [S1, unofficial]. It extends earlier work on linear depth cueing and hemispherical illumination by combining them into a single composite model [S1, unofficial]. The approach is based on the observation that the integrand can be expressed as a linear combination of Fourier projections, allowing the use of the projection-slice theorem for efficient rendering [S1, unofficial].

## Sources
[S1] PDF Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27 (unofficial)
[S2] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
