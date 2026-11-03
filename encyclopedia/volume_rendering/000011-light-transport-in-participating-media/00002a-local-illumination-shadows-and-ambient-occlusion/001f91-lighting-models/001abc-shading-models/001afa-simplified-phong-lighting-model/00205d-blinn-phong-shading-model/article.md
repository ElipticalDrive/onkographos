# Blinn-Phong Shading Model

Also called: Blinn-Phong Approximation, Blinn-Phong Local Illumination, Blinn-Phong Material Model, Blinn-Phong Shading, Blinn-Phong Specular, Blinn-Phong Specular Lighting

## Overview
The Blinn-Phong Shading Model is a local illumination model used to calculate specular highlights on surfaces, including those in volume rendering where the surface normal is approximated by the gradient of the scalar field [S1, S4]. It differs from the standard Phong model by using a half-vector between the light and view directions rather than the reflected light vector, which avoids the computationally expensive reflection vector calculation and provides smoother specular lobes [S2, S3]. It builds on [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) by introducing this half-vector approximation for the specular term.

## How it works
**Inputs:**
*   Position of the shading point $P$ (in world or texture space).
*   Surface normal $\mathbf{n}$ (normalized gradient vector for volume data).
*   Light position $\mathbf{l}_{pos}$ and light intensity/color $I_L$.
*   Camera/Eye position $\mathbf{e}_{pos}$.
*   Shininess exponent $n$ (controls specular lobe width).
*   Material coefficients: ambient $k_a$, diffuse $k_d$, specular $k_s$.
*   Base color/transfer function output $C_{base}$.

**Outputs:**
*   Final pixel color $C_{final}$.

**Data Structures:**
*   Vectors $\mathbf{n}, \mathbf{l}, \mathbf{v}, \mathbf{h}$ are 3D floating-point vectors.
*   Scalars $n, k_a, k_d, k_s, I_L$ are floating-point values.

**Steps:**
1.  **Coordinate Transformation:** If the light position is given in object space, convert it to texture/world space consistent with the shading point $P$ [S3].
    $$ \mathbf{l}_{pos}^{tex} = \frac{\mathbf{l}_{pos}^{obj} - \mathbf{offset}}{\mathbf{scale}} $$
    [G] Typical offset/scale for normalized texture coordinates $[-0.5, 0.5]$ is offset $(-0.5, -0.5, -0.5)$ and scale $(1.0, 1.0, 1.0)$ if mapping to $[0,1]$, or simply subtracting center if using $[-0.5, 0.5]$ directly.
2.  **Direction Vectors:** Compute the normalized light direction $\mathbf{l}$ and view direction $\mathbf{v}$ from the shading point $P$ [S1, S3].
    $$ \mathbf{l} = \text{normalize}(\mathbf{l}_{pos} - P) $$
    $$ \mathbf{v} = \text{normalize}(\mathbf{e}_{pos} - P) $$
    [Note: Some implementations define $\mathbf{v}$ as pointing towards the eye, others away. The dot product $\mathbf{n} \cdot \mathbf{l}$ requires $\mathbf{l}$ to point from surface to light. The half-vector definition $\mathbf{h} = \text{normalize}(\mathbf{l} + \mathbf{v})$ assumes both point away from the surface or both towards, depending on convention. Standard Blinn-Phong uses vectors pointing *away* from the surface for $\mathbf{l}$ and $\mathbf{v}$, or *towards* for both. The source [S3] uses $\mathbf{L} = \text{normalize}(L_{pos} - P)$ and $\mathbf{V} = \text{normalize}(-V_{dir})$, implying $\mathbf{V}$ points towards the eye. The half vector is $\mathbf{H} = \text{normalize}(\mathbf{L} + \mathbf{V})$. This is consistent if $\mathbf{L}$ and $\mathbf{V}$ are both defined as directions from the surface.]
3.  **Half Vector:** Compute the normalized half-vector $\mathbf{h}$ [S2, S3].
    $$ \mathbf{h} = \text{normalize}(\mathbf{l} + \mathbf{v}) $$
4.  **Diffuse Term:** Calculate the diffuse lighting component [S3].
    $$ D = \max(0, \mathbf{n} \cdot \mathbf{l}) $$
5.  **Specular Term:** Calculate the specular lighting component using the half-vector [S2, S3].
    $$ S = \begin{cases} \max(0, \mathbf{n} \cdot \mathbf{h})^n & \text{if } D > 0 \\ 0 & \text{otherwise} \end{cases} $$
    [G] The condition $D > 0$ ensures the specular highlight is only visible on the lit side of the surface, preventing artifacts when the half-vector points into the surface but the light is behind it.
6.  **Intensity Calculation:** Combine ambient, diffuse, and specular terms [S3, S4].
    $$ I = k_a I_a + (k_d D + k_s S) I_L $$
    Where $I_a$ is ambient light intensity.
7.  **Final Color:** Modulate the base color (from transfer function or material tint) by the calculated intensity [S1, S3, S4].
    $$ C_{final} = C_{base} \times I $$
    [Note: Source [S1] multiplies intensity by a tint color directly, effectively setting $k_d=k_s=1$ and $I_L=1$ for the tint multiplication, or treating tint as the base color modulated by the specular-only power. Source [S3] multiplies the full lighting intensity by the base RGB. Source [S4] states the computed intensity modulates the color components from the transfer function.]

**Parameters:**
*   $n$ (shininess): Controls the width of the specular highlight. Higher values produce sharper highlights. Typical values range from 1 to 1000+. Source [S1] uses 0.5, which is unusually low and likely a normalized or specific implementation parameter, not the standard exponent [S1, S2]. Source [S2] passes `exponent` via `obj->getExponent()`.

**Pitfalls:**
*   **Homogeneous Regions:** In volume rendering, the gradient (used as normal) is not well-defined in homogeneous regions, leading to noise or incorrect shading [S4].
*   **Solution:** Modify the model to shade only regions with high gradient magnitudes [S4].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per shading point. It involves a few vector normalizations, dot products, and a power function.
*   **Memory Complexity:** $O(1)$ auxiliary memory.
*   **Strengths:** Computationally inexpensive; provides visually plausible specular highlights; widely supported in hardware (fragment shaders).
*   **Limitations:** Local illumination only (no shadows, no indirect lighting); specular highlights can appear too broad or sharp depending on $n$; not physically based (energy conservation is not guaranteed without normalization factors like $\frac{n+2}{2\pi}$ [S2]).
*   **When to use:** Real-time volume rendering where performance is critical and approximate specular highlights are desired.
*   **When not to use:** When physically accurate light transport, shadows, or global illumination are required.

## Background
The Blinn-Phong model was introduced by James F. Blinn as an improvement to the Phong shading model [S4]. It is a standard local illumination model used in computer graphics, including volume rendering, to approximate specular reflections [S4]. In volume rendering, it is frequently applied by using the normalized gradient of the scalar field as the surface normal [S4]. The model is often modified in volume rendering contexts to handle the lack of well-defined normals in homogeneous regions [S4].

## Sources

[S1] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S4] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
