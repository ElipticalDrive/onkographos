# Blinn-Phong Toon Shading

Also called: Toon Blinn-Phong

## Overview
Blinn-Phong Toon Shading is a non-photorealistic illumination model that applies the Blinn-Phong lighting equations followed by aggressive quantization (posterization) of the diffuse component and binary thresholding of the specular component [S1, unofficial]. It distinguishes itself from standard Blinn-Phong by replacing smooth gradients with discrete color bands, and from other toon methods like [Volumetric Toon Shading](5276-volumetric-toon-shading.md) by retaining the specific mathematical structure of the Blinn-Phong half-vector specular term before applying the hard cut [S1, unofficial].

## How it works
**Inputs:** Surface normal $\mathbf{N}$, light direction $\mathbf{L}$, view direction $\mathbf{V}$, base color $C_{base}$, and shininess exponent $k$ (shininess).
**Outputs:** Final shaded color $C_{final}$.

**Data Structures:**
No complex data structures are required; the method operates per-fragment or per-voxel using scalar lighting factors.

**Steps:**
1.  **Calculate Diffuse Factor:** Compute the unclamped dot product of the light direction and surface normal, clamped to $[0, 1]$.
    $$ d_{raw} = \max(0, \mathbf{L} \cdot \mathbf{N}) $$
2.  **Calculate Specular Factor:** Compute the half-vector $\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|}$, then compute the dot product with the normal, raised to the shininess power.
    $$ s_{raw} = \max(0, \mathbf{H} \cdot \mathbf{N})^k $$
3.  **Posterize Diffuse Factor:** Apply discrete thresholds to $d_{raw}$ to create banding. The thresholds are $A=0.1$, $B=0.3$, $C=0.6$, and $D=1.0$ [S1, unofficial].
    $$ d_{toon} = \begin{cases} 0.0 & \text{if } d_{raw} < 0.1 \\ 0.3 & \text{if } 0.1 \le d_{raw} < 0.3 \\ 0.6 & \text{if } 0.3 \le d_{raw} < 0.6 \\ 1.0 & \text{if } d_{raw} \ge 0.6 \end{cases} $$
4.  **Apply Harsh Cut to Specular:** Apply a step function to $s_{raw}$ with a threshold of $0.2$ [S1, unofficial].
    $$ s_{toon} = \begin{cases} 0.0 & \text{if } s_{raw} < 0.2 \\ 1.0 & \text{if } s_{raw} \ge 0.2 \end{cases} $$
5.  **Compute Final Color:** Combine the base color with the toon lighting factors. Assuming ambient light is handled separately or included in $C_{base}$, the diffuse and specular contributions are added.
    $$ C_{final} = C_{base} \cdot d_{toon} + C_{specular} \cdot s_{toon} $$
    Where $C_{specular}$ is typically a bright color (e.g., white or light gray) representing the specular highlight intensity.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blinn_phong_toon_shading(N, L, V, C_base, C_specular, shininess):
    # Normalize inputs if necessary
    N = normalize(N)
    L = normalize(L)
    V = normalize(V)
    
    # 1. Diffuse factor
    dot_LN = max(0.0, dot(L, N))
    
    # 2. Specular factor
    H = normalize(L + V)
    dot_HN = max(0.0, dot(H, N))
    specular_raw = pow(dot_HN, shininess)
    
    # 3. Posterize diffuse
    if dot_LN < 0.1:
        diffuse_toon = 0.0
    elif dot_LN < 0.3:
        diffuse_toon = 0.3
    elif dot_LN < 0.6:
        diffuse_toon = 0.6
    else:
        diffuse_toon = 1.0
        
    # 4. Harsh cut specular
    if specular_raw < 0.2:
        specular_toon = 0.0
    else:
        specular_toon = 1.0
        
    # 5. Final color
    # Note: Ambient term is not defined in the source steps, assumed included in C_base or zero
    color = C_base * diffuse_toon + C_specular * specular_toon
    
    return color
```

**Parameters:**
*   Shininess ($k$): Not specified in the source notes. [G] Typical values range from 16 to 128 for plastic-like surfaces.
*   Diffuse Thresholds: $A=0.1, B=0.3, C=0.6, D=1.0$ [S1, unofficial].
*   Specular Threshold: $0.2$ [S1, unofficial].

**Edge Cases:**
*   **Zero Vector:** If $\mathbf{L} + \mathbf{V}$ is zero (light and view are opposite), $\mathbf{H}$ is undefined. [G] Handle by setting specular to 0 or using a fallback direction.
*   **Precision:** The step functions are sensitive to floating-point precision near the thresholds. [G] Use epsilon comparisons if necessary, though the discrete nature of toon shading often makes this less critical than in photorealistic rendering.

## Subtopics

### Edge Enhancement Shading
A shading mode that enhances edges by darkening pixels where the surface normal is nearly perpendicular to the view direction [S1, unofficial]. It calculates the dot product between the view vector $\mathbf{V}$ and normal $\mathbf{N}$ as $dv = \mathbf{V} \cdot \mathbf{N}$. The edge value is computed as $ev = (1.0 - |dv|)^{0.3}$. If $ev \ge 0.1$, the color is mixed towards black using the formula: $mix(color.rgb, vec3(0), ((ev - 0.1) / (1.0 - 0.1))^6)$. The edge threshold $et$ is $0.1$ [S1, unofficial]. This is often combined with Blinn-Phong Toon Shading to outline silhouettes.

## Operating characteristics
**Time and Memory Complexity:** $O(1)$ per fragment/voxel. No additional memory beyond standard lighting buffers.
**Measured Performance:** Not given in the sources.
**Scaling:** Scales linearly with the number of fragments/voxels. Independent of scene complexity as it relies on local geometry (normal, light, view).
**Strengths:** Computationally cheap; produces distinct, stylized "cartoon" or "cel-shaded" appearance; easy to implement in shaders.
**Limitations:** Discontinuous lighting can cause flickering if normals change rapidly between adjacent fragments/voxels (aliasing); lacks the smooth gradients of photorealistic models; parameter tuning (thresholds) is required for different scenes.
**When to use:** Stylized volume visualization, educational visualizations, or artistic rendering where a non-photorealistic look is desired.
**When not to use:** Photorealistic rendering, scenes requiring subtle lighting transitions, or when performance is constrained by memory bandwidth rather than computation (though this method is very fast).

## Background
Origin: The method is derived from the Blinn-Phong reflection model, adapted for toon shading. The specific implementation details (thresholds and step functions) are documented in the unofficial source [S1, unofficial].
History: Toon shading techniques have been used in computer graphics since the 1990s to emulate hand-drawn animation styles. The Blinn-Phong variant specifically leverages the half-vector approach for specular highlights, which is more efficient than the Phong model, and applies quantization to achieve the toon effect.
Usage: Used in real-time rendering applications, including games and interactive visualizations, to achieve a stylized aesthetic.

## Sources
[S1] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
