# Lambertian Lighting in SDF

Also called: Diffuse Lighting Model, Dot Product Lighting, Lambert Shading, Lambertian Diffuse, Lambertian Lighting

## Overview
Lambertian Lighting in SDF computes the diffuse illumination of a surface by evaluating the dot product between the surface normal and the light direction [S1, S2, S3, unofficial]. It calculates diffuse lighting with an ambient component [S1, unofficial], clamping negative dot products to zero to prevent back-face illumination [S2, unofficial].

## How it works
**Inputs:**
- Surface position $\mathbf{p}$ in world space [G].
- Surface normal $\mathbf{n}$, normalized [S1, unofficial].
- Light direction $\mathbf{l}$, normalized (pointing from surface to light) [S1, unofficial].
- Light color $\mathbf{c}_{light}$ [G].
- Base material color $\mathbf{c}_{base}$ [G].
- Ambient intensity $k_a$ [G].
- Light intensity $I$ [G].

**Outputs:**
- Final pixel color $\mathbf{c}_{final}$ [G].

**Data Structures:**
No complex data structures are required beyond standard vector types (float3/vec3) for positions, normals, and colors [G].

**Steps:**
1. **Normal Normalization:** Ensure the surface normal $\mathbf{n}$ is normalized. If retrieved from a texture, apply `N = normalize(Ntex)` [S1, unofficial].
2. **Light Direction Normalization:** Ensure the light direction vector $\mathbf{l}$ is normalized upon construction: `direction = direction.Normalize()` [S2, unofficial].
3. **Diffuse Term Calculation:** Compute the dot product between the normal and light direction, clamping negative values to zero:
   $$ d = \max(0.0, \mathbf{n} \cdot \mathbf{l}) $$
   [S1, S2, S3, unofficial].
4. **Color Accumulation:** Combine the ambient term and the diffuse term, scaled by light intensity and color, then modulate by the base material color:
   $$ \mathbf{c}_{final} = \mathbf{c}_{base} \times (k_a + d \times I) \times \mathbf{c}_{light} $$
   [S1, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lambertian_lighting_sdf(p, n, p_light, c_base, c_light, k_a=0.2, I=1.0):
    # Step 1: Normalize normal
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
- **Negative Dot Product:** The $\max(0.0, \dots)$ clamp is essential to prevent back-face illumination [G].
- **Precision:** When using finite differences for normals, small perturbations can lead to noisy normals. Using a sufficiently small epsilon for finite differences is critical [G].

## Subtopics

### [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md)
This method extends sphere tracing by computing diffuse lighting on surfaces defined by Signed Distance Fields (SDFs). It solves the problem of rendering shaded geometry without explicit mesh topology by deriving surface normals via finite differences and applying a Lambertian lighting model in world space. Unlike [Sphere Tracing Basics](52-sphere-tracing-basics.md), which returns a flat base color on hit, this method integrates a light direction and color to produce a shaded result.

### [Phong Shading with Shadows](4356-phong-shading-with-shadows.md)
Phong Shading with Shadows computes the final pixel color by applying the Phong reflection model (ambient, diffuse, and specular components) to an SDF, modulated by shadow factors to simulate occlusion. It differs from its parent [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by adding specular highlights and shadow casting, and from its sibling [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md) by explicitly separating the shading model logic from the ray-marching traversal loop, allowing for complex relighting of neural or implicit surfaces.

### Visualization-Oriented Shading
A shading configuration using camera-aligned two-sided diffuse shading and optional Screen-Space Ambient Occlusion (SSAO) to improve surface readability, contrast, and perception of local depth in scientific visualization [S4]. For two-sided rendering, the surface normal is reversed for back-facing fragments before evaluating the diffuse term [S4]. Camera-aligned lighting places the light direction near the viewing direction to provide consistent illumination over view-facing structures [S4].

## Operating characteristics
**Time Complexity:** $O(1)$ per pixel for the shading calculation itself, assuming the surface position and normal are already known from the ray marching step. The overall complexity is dominated by the ray marching step, which is $O(N)$ where $N$ is the number of steps to reach the surface [G].
**Memory Complexity:** $O(1)$ additional memory per pixel for lighting calculations [G].
**Strengths:** Computationally inexpensive; provides physically plausible diffuse shading for implicit surfaces; easy to implement [G].
**Limitations:** Does not account for inter-reflections (global illumination); specular highlights are absent; sensitive to normal estimation errors from finite differences [G].
**When to Use:** When real-time performance is critical and only diffuse lighting is required; for simple implicit object rendering [G].
**When Not to Use:** When realistic global illumination or specular effects are needed; when high-precision normals are critical and finite difference noise is unacceptable [G].

## Background
Lambertian reflectance is a fundamental model in computer graphics, describing a perfectly diffuse surface that scatters light equally in all directions [G]. Its application to SDFs leverages the ability of SDFs to provide continuous surface representations, allowing for shading without explicit mesh data [G].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S3] Sphere Tracing and Ray Marching, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5-sphere-tracing-and-ray-marching, fetched 2026-10-27 (unofficial)
[S4] Visual tools for 3D plasma simulation: GPU-based isosurface extraction ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC13674866/, fetched 2026-10-30
