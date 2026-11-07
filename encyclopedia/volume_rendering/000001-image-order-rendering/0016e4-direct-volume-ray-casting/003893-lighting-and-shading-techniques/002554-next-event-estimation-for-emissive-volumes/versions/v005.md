# Next Event Estimation for Emissive Volumes

Also called: Emissive Media Rendering, Emissive Volume Rendering with Next Event Estimation (NEE), NEE for Emissive Volumes

## Overview
Next Event Estimation (NEE) for Emissive Volumes is a direct lighting estimation technique used within the front-to-back compositing pipeline of [Direct volume ray casting](19-direct-volume-ray-casting.md) to reduce noise in scenes containing volumetric light sources. Unlike standard volume rendering which relies on sparse sampling of emission along the ray, NEE explicitly samples points on emissive volume boundaries or internal structures to compute direct illumination contributions. It sets itself apart from siblings like [Hybrid Gradient Volume Raycasting with Lasso Masking](2059-hybrid-gradient-volume-raycasting-with-lasso-masking.md) by focusing on statistical importance sampling of light sources rather than geometric masking, and from [Illumination Caching](7548-illumination-caching.md) by computing lighting on-the-fly rather than precomputing it.

## How it works
**Inputs and Outputs:**
*   **Input:** A ray segment $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ traversing the volume, the volume density field $V(\mathbf{p})$, the emission coefficient $\epsilon(\mathbf{p})$, and the scene geometry defining the volume bounds.
*   **Output:** The direct lighting contribution $C_{direct}$ at the current sample point $\mathbf{p}_i$, which is combined with the shading model defined in [Lighting and Shading Techniques](8281-lighting-and-shading-techniques.md).

**Data Structures:**
*   **Ray Bounds:** Two scalar values $t_{entry}$ and $t_{exit}$ representing the intersection distances of the ray with the volume bounding shape.
*   **PDF State:** A probability density value $p(\vec{x})$ representing the volume sampling PDF, and $p(\vec{\omega})$ representing the solid angle PDF of the sampled light point.

**Method Steps:**
1.  **Determine Ray Bounds:** Calculate the entry point $t_{entry}$ and exit point $t_{exit}$ of the ray with respect to the volume's bounding shape. This requires a ray-shape intersection test [S1, unofficial].
2.  **Sample Light Point:** Select a point $\mathbf{x}_L$ on the emissive volume boundary or within the volume according to a sampling strategy (e.g., uniform over surface area or volume). Let $\mathbf{l} = \mathbf{x}_L - \mathbf{p}_i$ be the vector to the light point and $r = \|\mathbf{l}\|$ be the distance.
3.  **Compute Solid Angle PDF:** Calculate the probability density of the sampled direction $\vec{\omega} = \mathbf{l}/r$ in solid angle space, denoted $p(\vec{\omega})$ [S1, unofficial].
4.  **Convert to Volume PDF:** The corresponding volume PDF $p(\vec{x})$ is derived from the solid angle PDF. The relationship involves the differential element between volume and solid angle. The term $r^{-2}p(\vec{\omega})$ is identified as a non-normalisable PDF component because $\int_V r^{-2} p(\vec{\omega}) \mathrm{dV}$ does not converge to 1 in the standard manner [S1, unofficial].
5.  **Calculate MIS Weights:** Compute the Multiple Importance Sampling (MIS) weight to balance the volume sampling strategy and the light sampling strategy. The weight is computed as $\mathrm{MIS}(r^{-2}p(\vec\omega), p(\vec x))$ [S1, unofficial].
6.  **Evaluate Emission and Shading:**
    *   Evaluate the emission $\epsilon(\mathbf{x}_L)$ at the light point.
    *   Compute the attenuation (transmittance) from $\mathbf{p}_i$ to $\mathbf{x}_L$ by integrating the density along the segment. *Note: Integrating the volume PDF over the ray length for this step is significantly slower (20x-40x) due to required ray casts and raises OptiX warnings in CUDA modes* [S1, unofficial].
    *   Compute the geometric term and shading contribution using the normal $\mathbf{n}$ at $\mathbf{p}_i$ and the light direction $\mathbf{l}$, consistent with the Phong/Blinn-Phong model in [Lighting and Shading Techniques](8281-lighting-and-shading-techniques.md).
7.  **Accumulate Contribution:** Combine the emission, attenuation, geometric term, and MIS weight to obtain the direct lighting contribution $C_{direct}$.
8.  **Consistency Check:** Ensure the emission evaluation along the ray segment is consistent with the direction sampling used in the MIS calculation to avoid artifacts [S1, unofficial].

**Formulas:**
*   **MIS Weight:**
    $$ w = \frac{f(x)}{g(x) + h(x)} $$
    Where $f(x)$ is the PDF of the chosen sampling strategy and $g(x), h(x)$ are the PDFs of the available strategies. Specifically for NEE in volumes, the balance involves $r^{-2}p(\vec\omega)$ and $p(\vec x)$ [S1, unofficial].
*   **Lighting Contribution:**
    $$ C_{direct} = \frac{\epsilon(\mathbf{x}_L) \cdot T(\mathbf{p}_i \to \mathbf{x}_L) \cdot (\mathbf{n} \cdot \mathbf{l}) \cdot k_d}{p(\vec{x})} \cdot w $$
    Where $T$ is the transmittance, $k_d$ is the diffuse coefficient, and $w$ is the MIS weight.

**Parameters:**
*   **Ray Bounds Precision:** Not given in the sources.
*   **MIS Heuristic:** Not given in the sources (standard balance heuristic is implied).

**Edge Cases and Pitfalls:**
*   **PDF Conversion Error:** Simply converting the solid angle PDF to a volume PDF using $r^{-2}\frac{\mathrm{dV}}{\mathrm{d\Omega}}$ yields incorrect results and is inconsistent with previous results obtained without NEE [S1, unofficial].
*   **Performance:** Integrating the volume PDF over the ray length is significantly slower (20x-40x) due to required ray casts and raises OptiX warnings in CUDA modes [S1, unofficial].
*   **Implementation Discrepancy:** In `llvm_*` modes, the implementation using NEE is approximately 5x slower than not using NEE. In `cuda_*` modes, it is approximately 1.5x slower [S1, unofficial].
*   **Backend Differences:** Ray entry and exit points may lead to different results between `llvm_*` (Embree) and `cuda_*` (OptiX) modes due to different handling of ray intersections [S1, unofficial].

## Subtopics

### Path Tracing with Next Event Estimation
A variant where NEE is integrated into a recursive path tracing framework for volumetric scenes. It recursively traces rays to compute pixel color, including direct lighting estimation via NEE. The base case returns black if depth is 0. It samples a point on the light, creates a shadow ray to check occlusion, and computes direct lighting using the formula: $(\text{attenuation}/255.0) \cdot \text{light\_color} \cdot \cos(\theta) / (\pi \cdot \text{pdf})$. Indirect lighting is computed recursively. This approach handles transmission materials by returning emitted light plus the scattered ray color modulated by transmittance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method introduces overhead due to ray-shape intersection tests for bounds and potential shadow ray casts for occlusion checking. In `llvm_*` modes, NEE is ~5x slower than non-NEE rendering. In `cuda_*` modes, it is ~1.5x slower [S1, unofficial].
*   **Memory Complexity:** Requires storing ray bounds and PDF states. No significant additional memory compared to standard volume rendering.
*   **Measured Performance:** Reduces noise for the same sample count, allowing for faster convergence in visually noisy areas [S1, unofficial].
*   **Strengths:** Significantly reduces noise in emissive volumes by directly sampling light sources.
*   **Limitations:** Higher computational cost per sample compared to standard volume emission evaluation. Implementation complexity regarding PDF consistency and backend differences (Embree vs OptiX).
*   **When to use:** Scenes with significant volumetric emission where noise is a primary concern.
*   **When not to use:** Real-time applications requiring high frame rates where the 1.5x-5x overhead is prohibitive, or when the volume is non-emissive.

## Background
The method originated from discussions and implementation efforts within the Mitsuba 3 renderer community, specifically in the context of adding emissive volume rendering capabilities. The core challenges regarding PDF conversion and MIS consistency were identified and discussed in GitHub issues and pull requests related to the `feature/emissive-media` branch [S1, unofficial]. The implementation aims to provide a robust way to handle emissive media that is consistent with path tracing principles.

## Sources
[S1] [✨ feature] Emissive Volume Rendering, https://github.com/mitsuba-renderer/mitsuba3/pull/678, fetched 2026-10-26 (unofficial)
[S2] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
