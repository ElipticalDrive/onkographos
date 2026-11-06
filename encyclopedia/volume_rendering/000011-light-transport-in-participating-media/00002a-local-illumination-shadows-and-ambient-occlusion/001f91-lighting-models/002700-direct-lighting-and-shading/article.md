# Direct Lighting and Shading

Also called: Direct Lighting, Lambertian Diffuse with GGX Specular, Lighting and Shading

## Overview
Direct Lighting and Shading computes the immediate illumination contribution of light sources at the surface of a volume-rendered object, typically during ray casting or splatting. It differs from its parent [Lighting Models](2056-lighting-models.md) by implementing a specific combination of Lambertian diffuse, ambient floor, emissive terms, and a non-standard GGX-like specular term [S1, unofficial]. Unlike [Shading models](837-shading-models.md) which may cover broader categories, this method focuses on the specific shading pipeline used in voxel-based real-time renderers, often including hard shadow testing via ray re-tracing [S1, unofficial].

## How it works
**Inputs:**
*   Intersection point $P$ and surface normal $\mathbf{n}$ (derived from volume gradient or voxel face).
*   Light direction $\mathbf{l}$ (normalized).
*   View direction $\mathbf{v}$ (normalized).
*   Material properties: albedo/color $C_{mat}$, emissive flag/intensity.
*   Light properties: color $C_{light}$, intensity.

**Outputs:**
*   Shaded color contribution $C_{shade}$.

**Data Structures:**
*   No complex data structures required; operates on scalar and vector values per intersection.

**Steps:**

1.  **Gradient/Normal Estimation:**
    If $\mathbf{n}$ is not provided by the traversal, compute it from the volume gradient $\mathbf{g} = \nabla V$ at $P$. Normalize to get $\mathbf{n} = \frac{\mathbf{g}}{\|\mathbf{g}\|}$. [S1, unofficial]

2.  **Diffuse Term Calculation:**
    Compute the Lambertian diffuse contribution using the dot product of the normal and light direction:
    $$ C_{diffuse} = \max(\mathbf{n} \cdot \mathbf{l}, 0) \cdot C_{light} $$
    [S1, unofficial]

3.  **Specular Term Calculation (Non-Standard GGX-like):**
    Compute the half-vector $\mathbf{h}$. **Note:** This implementation uses a non-standard definition:
    $$ \mathbf{h} = \text{normalize}(\mathbf{l} - \mathbf{v}) $$
    [S1, unofficial]
    *Pitfall:* The standard half-vector is $\mathbf{l} + \mathbf{v}$. Using $\mathbf{l} - \mathbf{v}$ may produce incorrect specular highlights [S1, unofficial].
    
    Compute the Fresnel approximation using Schlick's formula with base reflectivity $F_0$ (often 0.04 for dielectrics):
    $$ F = F_0 + (1 - F_0)(1 - \mathbf{n} \cdot \mathbf{h})^5 $$
    [S1, unofficial]
    
    The specular contribution is:
    $$ C_{specular} = F \cdot C_{light} $$
    [S1, unofficial]

4.  **Emissive Contribution:**
    If the voxel block is marked as emissive (determined by hash functions like `hash13` or `hash12` on block coordinates), add the emissive intensity $C_{emissive}$:
    $$ C_{emissive\_term} = \begin{cases} C_{emissive} & \text{if BLOCK\_EMISSIVE} \\ 0 & \text{otherwise} \end{cases} $$
    [S1, unofficial]

5.  **Ambient Floor:**
    Add a constant ambient term $C_{ambient}$ to prevent black shadows, often combined with the diffuse term or added separately. [S1, unofficial]

6.  **Shadow Testing (Optional):**
    If shadows are enabled, trace a ray from $P$ in direction $\mathbf{l}$ towards the light source. If the ray hits a non-empty voxel before reaching the light, set $C_{light} = 0$ for direct terms. [S1, unofficial]

7.  **Tone Mapping:**
    Apply exponential tone mapping to the final lit color $C_{lit}$:
    $$ C_{final} = 1 - \exp(-2 \cdot C_{lit}) $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_direct_lighting(n, l, v, light_color, material_color, is_emissive, emissive_color, enable_shadows, trace_ray):
    # 1. Diffuse
    diff_factor = max(dot(n, l), 0.0)
    diffuse = diff_factor * light_color
    
    # 2. Specular (Non-standard half-vector)
    h = normalize(l - v) # Note: l - v, not l + v
    f0 = 0.04 # Dielectric base reflectivity
    fresnel = f0 + (1.0 - f0) * pow(1.0 - dot(n, h), 5.0)
    specular = fresnel * light_color
    
    # 3. Emissive
    emissive = emissive_color if is_emissive else vec3(0.0)
    
    # 4. Shadow (if enabled)
    shadow_factor = 1.0
    if enable_shadows:
        if trace_ray(p, l): # Returns true if hit
            shadow_factor = 0.0
            
    # 5. Combine
    lit_color = (diffuse + specular) * shadow_factor + emissive
    
    # 6. Tone Mapping
    final_color = 1.0 - exp(-2.0 * lit_color)
    
    return final_color
```

## Subtopics

### [Physically-Based Rendering (PBR) Lighting](6815-physically-based-rendering-pbr-lighting.md)
Calculates direct lighting using a physically-based shading model with Burley diffuse and microfacet specular components, driven by metallic-roughness inputs. It differs from the parent method by replacing the non-standard GGX-like specular term and simple Lambertian diffuse with energy-conserving, view-dependent shading models.

### Split-Sum IBL Shading
A forward shading strategy where each surfel is shaded independently before rasterization using split-sum Image-Based Lighting (IBL), producing a linear HDR color value that enters the splatting pipeline [S2, unofficial]. The outgoing radiance $c_k$ is computed as:
$$ c_k = k_d \rho L_d(\mathbf{n}) + L_s(\mathbf{r}, r) (F_0 \beta_1 + \beta_2) $$
where $L_d$ is diffuse irradiance from a pre-convolved environment map, $L_s$ is prefiltered specular radiance, $F_0 = 0.04(1-\rho) + \rho^2$ is base reflectivity, and $k_d = (1-F_0)(1-\rho)$ ensures energy conservation [S2, unofficial].

### Anisotropic Specular Highlights (Disney)
A method for generating anisotropic specular highlights using the Disney roughness-to-alpha convention [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per intersection point, assuming constant-time light queries and shadow ray tracing. Shadow testing adds an additional voxel traversal cost.
*   **Memory Complexity:** $O(1)$ auxiliary memory.
*   **Strengths:** Fast, suitable for real-time rendering; simple to implement.
*   **Limitations:** The non-standard half-vector $\mathbf{l}-\mathbf{v}$ is physically incorrect for standard microfacet models [S1, unofficial]. Exponential tone mapping is a simple approximation. Ambient floor can reduce contrast.
*   **When to use:** Real-time voxel rendering where performance is critical and physical accuracy of specular highlights is secondary to style.
*   **When not to use:** Physically-based offline rendering requiring accurate energy conservation and correct microfacet behavior.

## Background
This method originates from real-time voxel world shaders, specifically those found in community-driven GLSL projects like the Voxel-World-Shader [S1, unofficial]. It is characterized by its specific implementation details, such as the non-standard half-vector and exponential tone mapping, which distinguish it from standard PBR pipelines. It is commonly used in voxel-based games and visualizations.

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S3] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
