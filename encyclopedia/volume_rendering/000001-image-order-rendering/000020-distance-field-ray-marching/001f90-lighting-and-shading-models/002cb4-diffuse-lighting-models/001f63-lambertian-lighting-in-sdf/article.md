# Lambertian Lighting in SDF

Also called: Diffuse Lighting Model, Dot Product Lighting, Lambert Shading, Lambertian Diffuse, Lambertian Lighting

## Overview
Lambertian Lighting in SDF computes the diffuse illumination of a surface defined by a Signed Distance Field (SDF) by evaluating the dot product between the surface normal and the light direction [S1, S2, S3, unofficial]. It solves the problem of shading implicit geometry without explicit mesh topology by deriving normals via finite differences and applying a standard diffuse reflectance model [S1, unofficial]. Unlike [Sphere Tracing Basics](52-sphere-tracing-basics.md), which returns a flat base color on hit, this method integrates light direction and color to produce a shaded result [S1, unofficial]. It differs from [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) and [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) by excluding specular highlights, focusing solely on the diffuse component [S2, unofficial].

## How it works
**Inputs:**
- Surface position $\mathbf{p}$ in world space.
- Surface normal $\mathbf{n}$, normalized.
- Light direction $\mathbf{l}$, normalized (pointing from surface to light).
- Light color $\mathbf{c}_{light}$ (typically RGB).
- Base material color $\mathbf{c}_{base}$ (typically RGB).
- Ambient intensity $k_a$ (scalar, typically $0.1$ to $0.5$).
- Light intensity $I$ (scalar, typically $1.0$).

**Outputs:**
- Final pixel color $\mathbf{c}_{final}$.

**Data Structures:**
No complex data structures are required beyond standard vector types (float3/vec3) for positions, normals, and colors.

**Steps:**
1. **Normal Normalization:** Ensure the surface normal $\mathbf{n}$ is normalized. If derived from an SDF gradient $\nabla f(\mathbf{p})$, compute $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [G].
2. **Light Direction Normalization:** Ensure the light direction vector $\mathbf{l}$ is normalized. If derived from a light position $\mathbf{p}_{light}$ and surface position $\mathbf{p}$, compute $\mathbf{l} = \frac{\mathbf{p}_{light} - \mathbf{p}}{\|\mathbf{p}_{light} - \mathbf{p}\|}$ [S1, S2, unofficial].
3. **Diffuse Term Calculation:** Compute the dot product between the normal and light direction, clamping negative values to zero to prevent back-face illumination:
   $$ d = \max(0.0, \mathbf{n} \cdot \mathbf{l}) $$
   [S1, S2, S3, unofficial].
4. **Color Accumulation:** Combine the ambient term and the diffuse term, scaled by light intensity and color, then modulate by the base material color:
   $$ \mathbf{c}_{final} = \mathbf{c}_{base} \times (k_a + d \times I) \times \mathbf{c}_{light} $$
   [S1, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lambertian_lighting_sdf(p, n, p_light, c_base, c_light, k_a=0.2, I=1.0):
    # Step 1: Normalize normal (assuming n might not be perfectly normalized if from gradient)
    n = n / length(n)
    
    # Step 2: Calculate and normalize light direction
    l = p_light - p
    l = l / length(l)
    
    # Step 3: Calculate diffuse term (clamped dot product)
    diff = max(dot(n, l), 0.0)
    
    # Step 4: Accumulate color
    # Ambient contribution
    ambient = k_a
    # Diffuse contribution
    diffuse = diff * I
    # Total illumination factor
    illumination = ambient + diffuse
    
    # Final color
    c_final = c_base * illumination * c_light
    
    return c_final
```

**Edge Cases and Pitfalls:**
- **Zero-Length Light Vector:** If the light position coincides with the surface position, the light direction vector becomes zero-length. A guard must check for this and handle it (e.g., default to a uniform light direction or skip lighting) [G].
- **Negative Dot Product:** The $\max(0.0, \dots)$ clamp is essential. Without it, back-facing surfaces would be illuminated, breaking the visual consistency of the SDF surface [S2, S3, unofficial].
- **Precision:** When using finite differences for normals, small perturbations can lead to noisy normals. Using a sufficiently small epsilon for finite differences is critical [G].

## Subtopics

### [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md)
This method extends sphere tracing by computing diffuse lighting on surfaces defined by SDFs. It solves the problem of rendering shaded geometry without explicit mesh topology by deriving surface normals via finite differences and applying a Lambertian lighting model in world space. Unlike [Sphere Tracing Basics](52-sphere-tracing-basics.md), which returns a flat base color on hit, this method integrates a light direction and color to produce a shaded result.

### [Phong Shading with Shadows](4356-phong-shading-with-shadows.md)
Phong Shading with Shadows computes the final pixel color by applying the Phong reflection model (ambient, diffuse, and specular components) to an SDF, modulated by shadow factors to simulate occlusion. It differs from its parent [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by adding specular highlights and shadow casting, and from its sibling [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md) by explicitly separating the shading model logic from the ray-marching traversal loop, allowing for complex relighting of neural or implicit surfaces.

### Visualization-Oriented Shading
A shading configuration using camera-aligned two-sided diffuse shading and optional Screen-Space Ambient Occlusion (SSAO) to improve surface readability, contrast, and perception of local depth in scientific visualization [S4]. For two-sided rendering, the surface normal is reversed for back-facing fragments before evaluating the diffuse term [S4]. Camera-aligned lighting places the light direction near the viewing direction to provide consistent illumination over view-facing structures [S4].

## Operating characteristics
**Time Complexity:** $O(1)$ per pixel for the shading calculation itself, assuming the surface position and normal are already known from the ray marching step. The overall complexity is dominated by the ray marching step, which is $O(N)$ where $N$ is the number of steps to reach the surface.
**Memory Complexity:** $O(1)$ additional memory per pixel for lighting calculations.
**Strengths:** Computationally inexpensive; provides physically plausible diffuse shading for implicit surfaces; easy to implement.
**Limitations:** Does not account for inter-reflections (global illumination); specular highlights are absent; sensitive to normal estimation errors from finite differences.
**When to Use:** When real-time performance is critical and only diffuse lighting is required; for simple implicit object rendering.
**When Not to Use:** When realistic global illumination or specular effects are needed; when high-precision normals are critical and finite difference noise is unacceptable.

## Background
Lambertian reflectance is a fundamental model in computer graphics, describing a perfectly diffuse surface that scatters light equally in all directions [S1, S2, S3, unofficial]. Its application to SDFs leverages the ability of SDFs to provide continuous surface representations, allowing for shading without explicit mesh data [S1, unofficial]. The method is widely used in ray marching tutorials and implicit surface rendering [S3, unofficial].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S3] Sphere Tracing and Ray Marching, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5-sphere-tracing-and-ray-marching, fetched 2026-10-27 (unofficial)
[S4] Visual tools for 3D plasma simulation: GPU-based isosurface extraction ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC13674866/, fetched 2026-10-30
