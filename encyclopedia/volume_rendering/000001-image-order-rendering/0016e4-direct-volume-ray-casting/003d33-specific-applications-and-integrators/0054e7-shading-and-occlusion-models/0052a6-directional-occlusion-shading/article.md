# Directional Occlusion Shading

Also called: Directional Occlusion, Directional Occlusion Shading Model, Schott et al, Slice-based directional occlusion shading

## Overview
Directional Occlusion Shading (DOS) is a slice-based shading technique for interactive direct volume rendering that approximates ambient occlusion and first-order scattering by integrating a backward-peaked cone phase function during front-to-back slice traversal [S5]. Unlike standard direct volume ray casting which uses simple front-to-back compositing, DOS modulates the contribution of each slice by an occlusion factor derived from the transmittance of neighboring voxels within a viewer-oriented cone, providing perceptual depth cues for both solid and transparent features without precomputation [S5]. It sets itself apart from sibling methods like [ISMOS](15205-ismos.md) by using a specialized cone phase function to accentuate semi-transparent surface-like features rather than approximating translucency, and from [Directional Filtering for Occlusion Reduction](14417-directional-filtering-for-occlusion-reduction.md) by computing occlusion interactively per-slice rather than filtering the final image [S5].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume dataset, 2D transfer function (mapping scalar values to color and opacity), camera parameters (view direction $\vec{\omega}$, projection matrix $P$), aperture angle $\theta$, inter-slice distance $d$, and sampling resolution for the occlusion cone [S5].
*   **Output:** Final color and opacity image in the frame buffer [S5].

**Data Structures:**
*   **Eye Buffer:** A floating-point buffer (typically 16-bit precision) storing the accumulated radiance $L$ and opacity for the current slice compositing [S5].
*   **Occlusion Buffers:** Two floating-point buffers (current and next) storing the fractional visibility (occlusion factor) $V$ for each slice. These are swapped after each slice update [S5].
*   **Proxy Geometry:** View-aligned slices (planes) intersecting the volume bounding box, computed on the CPU [S5].

**Numbered Steps:**
1.  **Compute Proxy Geometry:** On the CPU, generate view-aligned slices by intersecting planes with the volume bounding box [S5].
2.  **Initialize Buffers:** Allocate two floating-point occlusion buffers and one eye buffer. Initialize them to zero or background values [G].
3.  **Render Loop (for each slice from front to back):**
    a.  **Update Eye Buffer:** Render the current slice into the eye buffer. For each fragment, read the occlusion factor $V$ from the *current* occlusion buffer at the projected position. Compute the slice's contribution: $C_{slice} = \text{Color} \cdot V \cdot \text{AmbientRadiance}$. Composite this into the eye buffer using front-to-back alpha blending [S5].
    b.  **Update Occlusion Buffer:** Render the current slice again, but this time into the *next* occlusion buffer. For each fragment, compute the new occlusion factor by sampling the *current* occlusion buffer [S5].
        i.  Calculate occlusion extent: $r = d \tan(\theta)$ [S5].
        ii. Compute correction factor $\vec{c}$ in the vertex shader using projection matrix $P$ and view space z-component to handle perspective distortion [S5].
        iii. Generate sample positions $\vec{p}$ on a uniform grid disk of radius $r$ [S5].
        iv. Project fragment clip space position to NDC $\vec{x}_{y,d}$ via perspective division [S5].
        v.  Calculate NDC sample position: $\vec{p}_d = \vec{x}_{y,d} + \vec{c} \otimes \vec{p}$, where $\otimes$ is component-wise multiplication [S5].
        vi. Calculate texture coordinates: $\vec{p}_t = 0.5 \cdot \vec{p}_d + [0.5, 0.5]^T$ [S5].
        vii. Read values from the current occlusion buffer at $\vec{p}_t$. Attenuate each sample by the transmittance $T$ (approximated using distance $d$) and average the weighted samples to get the new occlusion factor for the next buffer [S5].
    c.  **Swap Buffers:** Swap the current and next occlusion buffers [S5].
4.  **Final Output:** Copy the eye buffer to the frame buffer, converting from floating-point to fixed-point [S5].

**Formulas:**
The radiance $L(\vec{x}, \vec{\omega})$ at a point $\vec{x}$ is given by:
$$ L(\vec{x}, \vec{\omega}) = T(\vec{x}, \vec{x}_0) L_b(\vec{x}_0, \vec{\omega}) + L_m(\vec{x}, \vec{\omega}) $$
where $L_b$ is background radiance, $T$ is transmittance, and $L_m$ is medium radiance [S5].

Medium radiance $L_m$ is approximated by considering only first-order scattering and ambient light $L_a$:
$$ L_m(\vec{x}, \vec{\omega}) = \int_{\vec{x}}^{\vec{x}_0} T(\vec{x}, \vec{x}') \sigma_s(\vec{x}') L_i(\vec{x}', \vec{\omega}) d\vec{x}' $$
$$ L_i(\vec{x}', \vec{\omega}) = L_a V(\vec{x}', \vec{\omega}) $$
where $\sigma_s$ is the scattering coefficient and $V$ is the fractional visibility (occlusion factor) [S5].

The fractional visibility $V$ is computed using a backward-peaked cone phase function $\Phi_\theta$:
$$ V(\vec{x}', \vec{\omega}) = \int_{4\pi} T(\vec{x}', \vec{x}_{0,i}) \Phi_\theta(\vec{\omega}, \vec{\omega}_i) d\vec{\omega}_i $$
The phase function is defined as:
$$ \Phi_\theta(\vec{\omega}, \vec{\omega}_i) = \begin{cases} \frac{1}{2\pi(1-\cos \theta)} & \text{if } \langle \vec{\omega}, \vec{\omega}_i \rangle > \cos(\theta) \\ 0 & \text{otherwise} \end{cases} $$
where $\theta \in [0, \pi/2)$ is the aperture angle, $\vec{\omega}$ is the viewing direction, and $\vec{\omega}_i$ are directions within the cone [S5].

Transmittance is $T(\vec{x}_a, \vec{x}_b) = e^{-\tau(\vec{x}_a, \vec{x}_b)}$ with optical thickness $\tau(\vec{x}_a, \vec{x}_b) = \int_{\vec{x}_a}^{\vec{x}_b} \sigma_t(\vec{x}) d\vec{x}$ [S5].

**Parameters:**
*   **Aperture Angle $\theta$:** Controls the influence of features at different distances. Range $[0, \pi/2)$. Typical value: $80^\circ$ [S5].
*   **Inter-slice Distance $d$:** Distance between view-aligned slices. Typical value: $0.001$ [S5].
*   **Sampling Rate/Grid Resolution:** Controls the quality of the occlusion integral approximation. Typical values tested: 0.3, 0.5, 1.0 [S3].

**Edge Cases and Pitfalls:**
*   **Perspective Distortion:** The cone base size varies with z-coordinate in screen space due to perspective projection. This is handled by the correction factor $\vec{c}$ computed in the vertex shader [S5].
*   **Inter-frame Artifacts:** Camera movement can cause artifacts. Mitigated by using a symmetric uniform grid for sample positions aligned with occlusion buffer axes [S5].
*   **Undefined OpenGL Behavior:** Reading and writing to the same layered render target (MRT) is undefined by the OpenGL spec but works in practice if reads and writes do not reference the same layer [S5].
*   **Light Source Limitation:** The method assumes a single light source located at the camera position (viewer-oriented) [S1].

## Subtopics

### [Extinction Optimized Shading](14805-extinction-optimized-shading.md)
Extinction Optimized Shading is a specific integrator mode within Directional Occlusion Shading that highlights interior structures occluded by others by calculating optimized extinction coefficients using a closed-form solution based on the transfer function's importance function. It sets itself apart from the standard directional occlusion mode by simplifying transfer function design to setting low or high importance, thereby revealing internal structures better than standard occlusion shading. It is chosen when the goal is to visualize internal details of complex, multivariate volumes where standard shading might hide occluded features.

## Operating characteristics
*   **Time Complexity:** Interactive. Performance depends on the number of slices, sampling rate, and GPU capabilities [S3, S5].
*   **Memory Complexity:** Low. Requires only two additional floating-point buffers for occlusion factors, similar to standard slice rendering [S1].
*   **Measured Performance:**
    *   Brain MRI (4 × 240 × 240 × 155, 600×600 viewport): 14 fps at sampling rate 0.3, 11 fps at 0.5, 4 fps at 1.0 [S3].
    *   Stag Beetle (1046 slices, 80° aperture): 10.6 FPS [S5].
    *   Brain MRI (256x256x160, 1000 slices): 3.7 FPS (DOS) vs 1.6 FPS (Lambertian) [S5].
    *   Carp CT (256x288x512, 220 slices): 48.3 FPS (DOS) vs 8.8 FPS (Lambertian) [S5].
    *   Visible Male (512x512x302, 962 slices): 5.0 FPS (DOS) vs 25.7 FPS (Lambertian) [S5].
    *   Engine Block CT (256x256x128, 679 slices): 5.6 FPS (DOS) vs 5.6 FPS (Lambertian) [S5].
    *   Hand CT (244x124x257, 619 slices): 13.3 FPS (DOS) vs 16.0 FPS (Lambertian) [S5].
    *   Tree CT (512x512x512, 1563 slices): 59.0 FPS (DOS) vs 29.1 FPS (Lambertian) [S5].
*   **Scaling:** Performance is generally better than Lambertian shading for complex volumes with many slices, but can be worse for simple volumes with few slices [S5]. MRT (Multiple Render Targets) implementation increases performance by 44-99% compared to SRT (Single Render Target), especially for low sample counts [S5].
*   **Strengths:** Interactive frame rates, no precomputation, supports interactive clipping planes and transfer function changes, provides good depth cues for transparent and solid features [S5].
*   **Limitations:** Bound to slice-based rendering, assumes light at camera position, may show regular structure artifacts [S5].

## Background
Directional Occlusion Shading was introduced by Mathias Schott, Vincent Pegoraro, Charles Hansen, Kévin Boulanger, and Kadi Bouatouch in 2009 at the Eurographics/ IEEE-VGTC Symposium on Visualization [S5]. It was published in the Journal of Computer Graphics Forum (Eurographics) in 2009 [S5]. The method is used in interactive direct volume rendering to provide perceptual cues similar to ambient occlusion without the cost of precomputation [S5].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S3] Continuous Indexed Points for Multivariate Volume Visualization, https://doi.org/10.26599/cvm.2025.9450496, fetched 2026-10-26
[S4] PDF A Comparison of Volumetric Illumination Methods by Considering their ..., https://ep.liu.se/ecp/094/005/ecp13094005.pdf, fetched 2026-10-28 (unofficial)
[S5] PDF A Directional Occlusion Shading Model for Interactive Direct Volume ..., https://cgl.ethz.ch/teaching/scivis_common/Literature/Schott2009.pdf, fetched 2026-10-29
