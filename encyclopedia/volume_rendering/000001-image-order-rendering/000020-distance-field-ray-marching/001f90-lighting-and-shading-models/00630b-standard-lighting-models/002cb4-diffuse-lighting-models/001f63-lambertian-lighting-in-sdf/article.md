# Lambertian Lighting in SDF

Also called: Diffuse Light, Diffuse Lighting Model, Dot Product Lighting, Lambert Shading, Lambertian Diffuse, Lambertian Lighting

## Overview
Lambertian Lighting in SDF calculates diffuse illumination for surfaces defined by Signed Distance Fields by computing the dot product between the surface normal and the light direction, clamping negative values to zero [S2, S4, unofficial]. It solves the problem of flat, unlit appearance in ray-marched scenes by providing physically plausible shading that reveals surface curvature and orientation [S3, unofficial]. Unlike [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) and [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md), which include specular highlights, this method computes only the diffuse component, resulting in a matte appearance [S2, unofficial][S5]. It builds on [Diffuse Lighting Models](5343-diffuse-lighting-models.md) by adapting the standard Lambertian reflectance model to the implicit geometry of SDFs, where normals are derived from the distance field gradient [G].

## How it works
**Inputs:**
- Surface point $\mathbf{p}$ in world or object space.
- Surface normal $\mathbf{n}$, normalized.
- Light direction $\mathbf{l}$, normalized (pointing from $\mathbf{p}$ to the light source).
- Base material color $\mathbf{c}_{base}$ (RGB).
- Light color $\mathbf{c}_{light}$ (RGB).
- Light intensity scalar $I_{light}$ (optional, often absorbed into $\mathbf{c}_{light}$).
- Ambient color $\mathbf{c}_{ambient}$ (optional, often a fraction of base color or light color).

**Outputs:**
- Final pixel color $\mathbf{c}_{final}$ (RGB).

**Data Structures:**
- Vectors $\mathbf{n}, \mathbf{l}$ are 3-component floating-point vectors.
- Colors $\mathbf{c}_{base}, \mathbf{c}_{light}, \mathbf{c}_{ambient}$ are 3-component floating-point vectors.

**Steps:**
1. **Compute Light Direction:** Determine the vector from the surface point $\mathbf{p}$ to the light source position $\mathbf{p}_{light}$. Normalize this vector to obtain $\mathbf{l}$.
   $$ \mathbf{l} = \frac{\mathbf{p}_{light} - \mathbf{p}}{\|\mathbf{p}_{light} - \mathbf{p}\|} $$
   If the light is directional, $\mathbf{l}$ is provided directly as a constant vector [S3, unofficial].

2. **Compute Diffuse Term:** Calculate the dot product of the surface normal $\mathbf{n}$ and the light direction $\mathbf{l}$. Clamp the result to a minimum of 0.0 to ensure no negative lighting contributions [S2, S3, S4, unofficial].
   $$ d = \max(0.0, \mathbf{n} \cdot \mathbf{l}) $$
   Some implementations include a strength factor $s$ (default 1.0) [S2, unofficial]:
   $$ d = \max(0.0, s \cdot (\mathbf{n} \cdot \mathbf{l})) $$

3. **Compute Final Color:** Combine the base color, ambient term, and diffuse contribution. The standard formula multiplies the base color by the sum of the ambient term and the product of the diffuse term, light intensity, and light color [S1, unofficial].
   $$ \mathbf{c}_{final} = \mathbf{c}_{base} \times (\mathbf{c}_{ambient} + d \times I_{light} \times \mathbf{c}_{light}) $$
   If ambient is not used, $\mathbf{c}_{ambient}$ is zero. If light intensity is not separated, it is included in $\mathbf{c}_{light}$ [S4, unofficial].

**Parameters:**
- Ambient factor: Not given in the sources. Often set to a small constant (e.g., 0.1) or derived from environment maps [G].
- Light Intensity: Not given in the sources. Typically 1.0 for basic tutorials [G].

**Edge Cases and Pitfalls:**
- **Back-facing surfaces:** In standard rendering, back-facing surfaces are culled. In SDF ray marching, rays may hit the "back" of a surface if the ray origin is inside or if two-sided rendering is enabled. The dot product $\mathbf{n} \cdot \mathbf{l}$ will be negative, and clamping to 0.0 results in no diffuse light, which is correct for opaque surfaces [S5].
- **Normal estimation error:** The normal $\mathbf{n}$ is estimated via finite differences of the SDF. This approximation can be noisy or inaccurate near singularities or very small features, leading to shading artifacts [G].
- **Light position conversion:** In some frameworks (e.g., Unity), light positions may need conversion from object space to texture or world space before computing the direction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lambertian_lighting_sdf(p, p_light, normal, base_color, light_color, ambient_color=vec3(0.0), light_intensity=1.0):
    """
    Computes Lambertian diffuse lighting for an SDF surface point.
    
    Args:
        p: vec3, surface point in world space.
        p_light: vec3, light source position in world space.
        normal: vec3, normalized surface normal at p.
        base_color: vec3, base material color.
        light_color: vec3, color of the light source.
        ambient_color: vec3, ambient light color. Default 0.
        light_intensity: float, intensity multiplier. Default 1.0.
        
    Returns:
        vec3, final pixel color.
    """
    # Step 1: Compute light direction
    # Handle directional lights if p_light is a direction vector (w=0) or similar convention
    # Assuming point light for this implementation
    light_dir = p_light - p
    light_dir_length = length(light_dir)
    
    # Avoid division by zero if point is exactly at light source
    if light_dir_length < 1e-6:
        light_dir = vec3(0.0, 1.0, 0.0) # Arbitrary default
    else:
        light_dir = light_dir / light_dir_length
        
    # Step 2: Compute diffuse term
    # Dot product of normal and light direction
    dot_product = dot(normal, light_dir)
    
    # Clamp to 0.0
    diffuse = max(dot_product, 0.0)
    
    # Step 3: Compute final color
    # Formula: base * (ambient + diffuse * intensity * light_color)
    # Note: Some sources multiply light_color by intensity, others absorb intensity into light_color.
    # We follow [S1, unofficial] structure: base * (ambient + diff * lightIntensity * lightColor)
    
    diffuse_contribution = diffuse * light_intensity * light_color
    total_light = ambient_color + diffuse_contribution
    final_color = base_color * total_light
    
    return final_color
```

## Subtopics

### [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md)
Sphere Tracing with Shading is a method for computing diffuse illumination on surfaces defined by Signed Distance Fields (SDFs) during the ray marching process. It extends basic sphere tracing by estimating surface normals via finite differences and applying a Lambertian lighting model in world space. Unlike [Sphere Tracing Basics](52-sphere-tracing-basics.md), which returns a flat base color, this method calculates per-pixel lighting intensity based on the angle between the surface normal and the light direction. It builds on [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by providing the specific implementation details for normal estimation and integration within the Unity shader pipeline.

### [Phong Shading with Shadows](4356-phong-shading-with-shadows.md)
Phong Shading with Shadows calculates the final color of a surface point in a volume rendering pipeline by combining ambient, diffuse, and specular lighting contributions using the Phong reflection model, modulated by a shadow factor. It solves the problem of providing physically plausible surface appearance and depth cues for SDF-based volume rendering by accounting for occlusion from light sources. Unlike its parent [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md), which only computes diffuse lighting, this method adds specular highlights and ambient terms to better define surface curvature and material properties. It differs from [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md) by explicitly supporting multiple light types (positional, head, camera) and configurable attenuation, whereas sphere tracing variants often focus on the marching algorithm itself.

### [Lambertian Diffuse Shading with Fog](15995-lambertian-diffuse-shading-with-fog.md)
This method computes the final pixel color by combining Lambertian diffuse reflection, ambient lighting, and exponential fog attenuation, then mapping the result to terminal colors with optional gamma correction. It extends [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by adding fog density and color mapping logic specific to ASCII/ANSI console rendering.

### Visualization-Oriented Shading
A shading configuration using camera-aligned two-sided diffuse shading and optional Screen-Space Ambient Occlusion (SSAO) to improve surface readability, contrast, and perception of local depth in scientific visualization [S5]. For two-sided rendering, the surface normal is reversed for back-facing fragments before evaluating the diffuse term. Camera-aligned lighting places the light direction near the viewing direction to provide consistent illumination over view-facing structures [S5].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel per light source, assuming the normal is already computed. The normal computation itself is $O(1)$ but requires multiple SDF evaluations (typically 4 for finite differences) [G].
- **Memory Complexity:** $O(1)$ auxiliary memory per pixel.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with the number of light sources.
- **Strengths:** Simple to implement; computationally inexpensive; provides clear visual cues for surface orientation and curvature; physically plausible for matte surfaces.
- **Limitations:** Does not account for specular highlights, making materials look uniformly matte; sensitive to normal estimation errors from finite differences; does not handle complex lighting effects like reflections or refractions.
- **When to use:** When rendering matte, non-shiny materials; when computational resources are limited; as a base layer for more complex shading models.
- **When not to use:** When rendering shiny, metallic, or glossy materials; when high-fidelity global illumination is required.

## Background
Lambertian reflectance is a fundamental model in computer graphics, describing ideal diffuse reflection where light is scattered equally in all directions [S3, unofficial]. In the context of SDFs, it was adapted for ray marching techniques to provide shading for implicit surfaces. The method is widely used in real-time rendering, including WebGL and Unity shader pipelines, as a standard way to add depth and form to ray-marched geometry [S1, S4, unofficial]. It is also used in scientific visualization to enhance the perception of surface orientation in isosurfaces [S5].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S3] Painting with Math: A Gentle Study of Raymarching, https://blog.maximeheckel.com/posts/painting-with-math-a-gentle-study-of-raymarching/, fetched 2026-10-27 (unofficial)
[S4] Sphere Tracing and Ray Marching, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5-sphere-tracing-and-ray-marching, fetched 2026-10-27 (unofficial)
[S5] Visual tools for 3D plasma simulation: GPU-based isosurface extraction ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC13674866/, fetched 2026-10-30
