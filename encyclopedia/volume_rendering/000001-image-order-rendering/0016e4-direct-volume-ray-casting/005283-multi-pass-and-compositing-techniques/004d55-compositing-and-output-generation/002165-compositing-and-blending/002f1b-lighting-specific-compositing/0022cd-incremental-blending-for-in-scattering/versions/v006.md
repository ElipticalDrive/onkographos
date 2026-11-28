# Incremental Blending for In-scattering


## Overview
Incremental Blending for In-scattering is a method for approximating light scattering (translucency) in volume rendering by accumulating indirect lighting components across multiple sampling directions [S1]. It solves the problem of computing in-scattering in a single-pass rendering pipeline by discretizing the scattering sphere into $J$ directions and blending the result of each direction into a stored accumulator [S1]. It differs from full scattering models by ignoring back-scattering during the intensity transport phase, which simplifies the hardware implementation at the cost of accuracy in dense, deep regions of the volume [S1]. It builds on [Lighting-Specific Compositing](5940-lighting-specific-compositing.md) by introducing indirect attenuation parameters and a dedicated light buffer for scattering propagation.

## How it works
**Inputs:**
*   Volume dataset $V$.
*   Transfer function mapping density to color $c(s)$ and opacity $\alpha(s)$.
*   Light source direction and intensity.
*   Number of scattering directions $J$.
*   Bias parameter $I_{bias} \in [0, 1]$ [S1].

**Outputs:**
*   Final eye color $I_{eye}$ for each pixel.

**Data Structures:**
*   $I_{stored}$: A buffer (volume or texture) storing the accumulated in-scattering intensity. Initialized to zero or a bias value.
*   $I_{final}$: A temporary buffer storing the result of the current direction's integration.
*   $g_k$: The emitted color term at sample $k$, computed per sample along the ray.

**Steps:**

1.  **Initialization:** Initialize $I_{stored}$ to zero. Determine $J$ sampling directions by subdividing a geometric shape (tetrahedron, icosahedron, or octahedron) [S1].

2.  **Direction Loop:** For each incremental update step $j$ from 1 to $J$:
    a.  **Ray Casting:** For each view ray, perform volume ray casting. At each sample point $s_k$ along the ray:
        i.  Compute the scattered intensity $I_s(s_k)$ for the current direction $j$. This involves calculating the light contribution from the light source direction, potentially using shadowing or attenuation [S1].
        ii. Compute the emitted color term $g_k$ using the formula:
        $$g_k = (I_{bias} + (1 - I_{bias}) \cdot I_s(s_k)) \cdot c(s_k) \cdot \alpha(s_k)$$
        where $c(s_k)$ is the color from the transfer function, $\alpha(s_k)$ is the opacity, and $I_{bias}$ adjusts the minimum light level [S1].
        iii. Compute the intermediate intensity $I_{final}$ for the current direction by integrating $g_k$ along the ray using the standard front-to-back compositing equation:
        $$I_{final} = \sum_{k=0}^n g_k \prod_{p=0}^{k-1} (1 - \alpha(s_p))$$
        where $n$ is the total number of samples [S1].
    b.  **Incremental Blend:** Blend the newly computed $I_{final}$ with the previously stored intensity $I_{stored}$ using the incremental blending formula:
        $$I_{blended} = \frac{1}{j} \cdot I_{final} + \left(1 - \frac{1}{j}\right) \cdot I_{stored}$$
        where $I_{blended}$ is the new value to store, $I_{final}$ is the result from the current direction $j$, and $I_{stored}$ is the previously accumulated intensity [S1].
    c.  **Store:** Update $I_{stored} \leftarrow I_{blended}$ [S1].

3.  **Final Output:** After all $J$ directions are processed, $I_{stored}$ contains the approximated in-scattering contribution. This is combined with direct lighting and other compositing terms to produce the final image [S1].

**Parameters:**
*   $J$: Number of scattering directions. Higher values improve accuracy but increase computation time [S1].
*   $I_{bias}$: Minimum light level parameter in range $[0, 1]$. Default behavior assumes a white light source [S1].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** The method requires storing intermediate results ($\alpha \cdot l_p$, $\alpha \cdot g$, $I_{final}$) for each voxel, which can be significant for large volumes (e.g., $512^3$) [S1]. Reducing the resolution of the scattering volume (SVR) can mitigate this [S1].
*   **Back-scattering:** The method ignores light scattered in the backward direction relative to the light source. This simplification leads to inaccuracies in dense, deep regions of the volume [S1].
*   **Light Source Color:** The base equations assume a white light source. Extension to colored light requires modifying the $I_s(s_k)$ calculation [S1].

## Subtopics

### [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md)
Incremental Lighting for In-Scattering is a method for approximating light scattering (translucency) in volume rendering by propagating and blurring indirect lighting components from slice to slice, considering only incoming light within a cone in the direction of the light source. It solves the problem of computing indirect illumination in a single-pass rendering pipeline by discarding backward-directed light relative to the light source, thereby accommodating the lack of knowledge about future slices during a front-to-back traversal [15652-forward-scattered-intensity-transport.md]. It differs from full scattering models by ignoring back-scattering during the intensity transport phase, which simplifies the hardware implementation at the cost of accuracy in dense, deep regions of the volume [15652-forward-scattered-intensity-transport.md]. It builds on [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md) by introducing indirect attenuation parameters and a dedicated light buffer for scattering propagation.

### [Hardware Blending Recurrence Evaluation](5317-hardware-blending-recurrence-evaluation.md)
Hardware Blending Recurrence Evaluation is a technique for evaluating the multiple forward scattering recurrence $I_k = g_{k-1} + T_{k-1} I_{k-1}$ by mapping the mathematical operation to GPU hardware blending and pixel read-back operations. It computes the incident light on each particle in a participating medium by using the frame buffer as an accumulator. It differs from [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md) by explicitly reading back pixel values to resolve the recurrence dependency.

### Volume Ray Casting with In-scattering
Volume Ray Casting with In-scattering is a specific application of the incremental blending method where the in-scattering term $g_k$ is computed directly during the ray casting pass. The final eye color $I_{eye}$ is computed as:
$$I_{eye} = \sum_{k=0}^n g_k \prod_{p=0}^{k-1} (1 - \alpha(s_p))$$
where $g_k$ is the emitted color at sample $k$, $\alpha(s_p)$ is opacity, and $n$ is total samples [S1]. The term $g_k$ is defined as:
$$g_k = (I_{bias} + (1 - I_{bias}) \cdot I_s(s_k)) \cdot c(s_k) \cdot \alpha(s_k)$$
where $I_{bias}$ is the minimum light level, $I_s(s_k)$ is the scattered intensity, and $c(s_k)$ is the color from the transfer function [S1]. The bias parameter $I_{bias}$ is in the range $[0..1]$ and adjusts the minimum light level, with default behavior assuming a white light source [S1].

### Incremental Integration for Epipolar Rendering
Incremental Integration for Epipolar Rendering is a method for computing volumetric shadows and scattering by sweeping over view rays $\beta$ from the light away. It maintains an incremental function $b[\gamma]$ storing $\Gamma(\gamma)V(\gamma, \beta)$. Initially $\beta=0$ and $b[\gamma]=\Gamma(\gamma)$. When stepping to a new $\beta$, $b[\gamma]$ is set to 0 for all light rays $\gamma$ where $S[\gamma]=\beta$ (light rays terminating at the current view ray). The integral is $B(\beta) \sum_{\gamma<D[\beta]} b[\gamma]$ [S2]. The intensity $I[\beta]$ is computed as:
$$I[\beta] = \sum_{\substack{\gamma < D[\beta] \\ S[\gamma] > \beta}} I(\gamma, \beta) V(\gamma, \beta) = \sum_{S[\gamma] > \beta} I(\gamma, \beta)$$
where $D[\beta]$ is the view ray length from the rectified depth map, $S[\gamma]$ is the blocker distance from the rectified shadow map, $I(\gamma, \beta)$ contains scattering terms excluding visibility, and $V(\gamma, \beta)$ is visibility (1 if $S[\gamma] > \beta$, 0 otherwise) [S2]. A pitfall of naive incremental summation is that it still requires iterating over all $\gamma$ for each $\beta$, resulting in $O(srd)$ complexity, same as brute force ray marching [S2].

## Operating characteristics
*   **Time Complexity:** The method requires $J$ passes over the volume, where $J$ is the number of scattering directions. Each pass involves ray casting and compositing. The complexity is $O(J \cdot N)$, where $N$ is the number of samples per ray [S1].
*   **Memory Complexity:** The method requires storage for intermediate volumes ($\alpha \cdot l_p$, $\alpha \cdot g$, $I_{final}$), which can be significant. For a $512^3$ volume, three additional $512^3$ volumes may be needed [S1]. Reducing the resolution of the scattering volume (SVR) can reduce memory usage [S1].
*   **Strengths:** It allows for the approximation of in-scattering in a single-pass pipeline without requiring complex global illumination algorithms. It is hardware-friendly due to the incremental blending approach [S1].
*   **Limitations:** It ignores back-scattering, leading to inaccuracies in dense, deep regions. The accuracy depends on the number of directions $J$. Memory usage can be high for large volumes [S1].
*   **When to use:** When real-time or near-real-time rendering of translucent volumes is required, and full global illumination is too expensive.
*   **When not to use:** When high accuracy in dense, deep regions is critical, or when back-scattering effects are visually significant.

## Background
The method was introduced in "Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration" [S1]. It builds on the principles of direct volume ray casting and compositing, adapting them to handle in-scattering effects incrementally. The approach was developed to address the computational cost of global illumination in volume rendering, providing a practical approximation for real-time applications.

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
