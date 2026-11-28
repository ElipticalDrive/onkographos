# Direct Lighting and Shading

Also called: Direct Lighting, Lambertian Diffuse with GGX Specular, Lighting and Shading

## Overview
Direct Lighting and Shading computes illumination at a point $\mathbf{p}$ within or on the boundary of a volume by evaluating contributions from discrete light sources and ambient terms, while approximating surface-like appearance through local shading models [S1, unofficial]. It differs from [Shading models](837-shading-models.md) by explicitly integrating direct light visibility and shadowing into the volume traversal context, and from [Cook-Torrance BRDF](2509-cook-torrance-brdf.md) by employing simplified or non-standard specular approximations (e.g., modified half-vectors) rather than strict microfacet theory [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Hit position $\mathbf{p}$, surface normal $\mathbf{n}$, light direction $\mathbf{l}$, view direction $\mathbf{v}$, material properties (albedo $\rho$, emissive flag), and light color $C_{light}$ [S1, unofficial].
*   **Outputs:** Final shaded color $C_{final}$.

**Data Structures:**
*   **Material Block:** Contains flags for emissive state (`BLOCK_EMISSIVE`) and material type, often determined via hash functions (e.g., `hash13`, `hash12`) on block coordinates in voxel implementations [S1, unofficial].
*   **Light Source:** Defined by direction $\mathbf{l}$ and color $C_{light}$ [S1, unofficial].

**Steps:**
1.  **Normal Approximation:** Compute the normal $\mathbf{n}$ from the volume gradient $\mathbf{g} = \nabla V$ at $\mathbf{p}$: $\mathbf{n} = \mathbf{g} / \|\mathbf{g}\|$ [G]. For immediate starting-voxel hits, a fallback normal of $(0, 1, 0)$ is used [S1, unofficial].
2.  **Diffuse Term:** Calculate the Lambertian diffuse contribution: $C_{diffuse} = \max(\mathbf{n} \cdot \mathbf{l}, 0) \cdot C_{light}$ [S1, unofficial].
3.  **Specular Term:** Compute the specular highlight using a non-standard half-vector $\mathbf{h}$ and Schlick Fresnel approximation:
    *   Compute half-vector: $\mathbf{h} = \text{normalize}(\mathbf{l} - \mathbf{v})$ [S1, unofficial].
    *   Compute Fresnel: $F = F_0 + (1 - F_0)(1 - \mathbf{n} \cdot \mathbf{h})^5$, where $F_0$ is the base reflectivity [S1, unofficial].
    *   Compute specular: $C_{specular} = F \cdot (\mathbf{n} \cdot \mathbf{h})^\alpha \cdot C_{light}$, where $\alpha$ is the shininess exponent [S1, unofficial].
4.  **Emissive Contribution:** If the block at $\mathbf{p}$ is marked `BLOCK_EMISSIVE` (determined by hash-based checks), add the emissive intensity $C_{emissive}$ to the total [S1, unofficial].
5.  **Shadowing:** Test for occlusion between $\mathbf{p}$ and the light source using a single hard occlusion check via voxel traversal from the hit point toward $\mathbf{l}$ [S1, unofficial]. If occluded, direct lighting contributions are skipped [S1, unofficial].
6.  **Ambient Term:** Add an ambient floor $C_{ambient}$ to prevent total blackness in shadows [S1, unofficial].
7.  **Tone Mapping:** Apply exponential tone mapping to the final lit color $C_{lit}$: $C_{final} = 1 - \exp(-2 \cdot C_{lit})$ [S1, unofficial].

**Parameters:**
*   Typical values for $\alpha$ (shininess exponent) and $F_0$ (base reflectivity) are not given in the sources.

**Edge Cases and Pitfalls:**
*   The half-vector is computed as $\mathbf{h} = \text{normalize}(\mathbf{l} - \mathbf{v})$ instead of the standard $\mathbf{l} + \mathbf{v}$, which may produce incorrect specular highlights [S1, unofficial].
*   The fallback normal $(0, 1, 0)$ is applied only to immediate starting-voxel hits and may be inaccurate for interior origins [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_direct_lighting(p, n, l, v, material, light_color, is_shadowed):
    # Step 1: Diffuse (Lambertian)
    ndotl = max(dot(n, l), 0.0)
    diffuse = ndotl * light_color

    # Step 2: Specular (Non-standard half-vector)
    h = normalize(l - v)
    ndoth = max(dot(n, h), 0.0)
    f0 = material.f0
    fresnel = f0 + (1.0 - f0) * pow(1.0 - ndoth, 5.0)
    specular = fresnel * pow(ndoth, material.shininess) * light_color

    # Step 3: Shadowing (Hard occlusion)
    if is_shadowed:
        diffuse = vec3(0.0, 0.0, 0.0)
        specular = vec3(0.0, 0.0, 0.0)

    # Step 4: Emissive
    emissive = vec3(0.0, 0.0, 0.0)
    if material.is_emissive:
        emissive = material.emissive_intensity

    # Step 5: Ambient
    ambient = material.ambient_floor

    # Step 6: Combine
    lit = diffuse + specular + emissive + ambient

    # Step 7: Tone Mapping
    final = 1.0 - exp(-2.0 * lit)

    return final
```

## Subtopics

### [Physically-Based Rendering (PBR) Lighting](6815-physically-based-rendering-pbr-lighting.md)
Calculates direct lighting using a physically-based shading model with Burley diffuse and microfacet specular components, driven by metallic-roughness inputs. It differs from the parent method by replacing the non-standard GGX-like specular term and Lambertian diffuse with energy-conserving, view-dependent shading. Choose it when physical accuracy and energy conservation are required.

### Split-Sum IBL Shading
A forward shading strategy that shades each surfel independently before rasterization using split-sum Image-Based Lighting to produce linear HDR color values for the splatting pipeline [S2, unofficial]. It differs from the parent method by using pre-convolved environment maps and BRDF LUTs for efficient indirect lighting approximation [S2, unofficial]. Choose it for surface splatting pipelines requiring efficient IBL [S2, unofficial].

### Anisotropic Specular Highlights (Disney)
Generates anisotropic specular highlights using the Disney roughness-to-alpha convention [S3, unofficial]. It differs from the parent method by explicitly modeling directional roughness for anisotropic reflections [S3, unofficial]. Choose it when stylized or directional specular highlights are needed [S3, unofficial].

### Sky Light Estimation with MIS
Uses Multiple Importance Sampling for environment sky light, applying MIS to both light and phase functions [S4, unofficial]. It differs from the parent method by handling continuous environment lighting distributions rather than discrete point/directional lights [S4, unofficial]. Choose it for accurate sky illumination in volumetric path tracers [S4, unofficial].

### Blinn-Phong Relighting
Uses the Blinn-Phong shading model to allow real-time lighting adjustments by scaling parameters $k_a$, $k_d$, $k_s$, and $\beta$ during inference [S5]. It differs from the parent method by providing explicit parameter scaling for dynamic relighting in Gaussian splatting visualizations [S5]. Choose it for interactive volume visualization requiring real-time lighting edits [S5].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per shading point for the lighting calculation itself, but shadowing requires additional ray traversal steps, potentially $O(N)$ where $N$ is the number of samples along the light ray [S1, unofficial].
*   **Memory Complexity:** $O(1)$ for local shading state. IBL variants require storage for environment maps and BRDF LUTs [S2, unofficial].
*   **Strengths:** Computationally efficient for real-time applications; provides basic shadowing and surface detail [S1, unofficial].
*   **Limitations:** The non-standard half-vector implementation in some voxel shaders produces incorrect specular highlights [S1, unofficial]. Ambient terms are often simplistic approximations [S1, unofficial].
*   **When to use:** Real-time voxel rendering where performance is critical and physical accuracy of specular highlights is secondary [S1, unofficial].
*   **When not to use:** Offline rendering requiring strict energy conservation or physically accurate specular reflections [S1, unofficial].

## Background
The method originates from real-time voxel rendering techniques, specifically those implemented in GLSL shaders for procedural voxel worlds [S1, unofficial]. It combines standard local illumination models with voxel-specific optimizations like hash-based block type determination and fast voxel traversal for shadowing [S1, unofficial]. The use of non-standard half-vectors suggests an adaptation for specific visual styles or performance constraints in early voxel engines [S1, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S3] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S4] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
[S5] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
