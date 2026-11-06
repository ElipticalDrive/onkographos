# Blinn-Phong with Metallic Reflections

Also called: Neon City Lighting, Wet Asphalt Shader

## Overview
A custom shading model for ray-marched signed distance fields (SDFs) that extends the classic Blinn-Phong approximation with environment reflections and emissive contributions to simulate urban night scenes [S1, unofficial]. It distinguishes itself from standard Blinn-Phong by modulating specular intensity, power, and reflection weight based on a material's metallic property, and from GGX microfacet models by using a simplified Schlick-like Fresnel term rather than a full microfacet distribution [S1, unofficial].

## How it works
**Inputs:**
- Surface point $\mathbf{p}$ in world space.
- Ray direction $\mathbf{rd}$ (from camera to $\mathbf{p}$, normalized).
- Material structure `sf` containing:
  - `albedo`: `vec3` base color.
  - `metal`: `float` in $[0, 1]$ (0 = non-metal, 1 = metal).
  - `emissive`: `vec3` self-illuminated color.
- Scene functions: `calcNormal(p)`, `softShadow(p, l, radius)`, `calcAO(p, n)`, `skyColor(r)`.

**Outputs:**
- Final color `vec3`.

**Data Structures:**
The material structure `sf` is determined by a material ID derived from the SDF evaluation (e.g., via hashing the cell coordinates or distance field value).

**Steps:**
1.  **Normal Calculation:** Compute the surface normal $\mathbf{n}$ at $\mathbf{p}$.
    $$ \mathbf{n} = \text{calcNormal}(\mathbf{p}) $$
    [G] Typically estimated via central finite differences of the SDF $f(\mathbf{x})$:
    $$ \mathbf{n} = \frac{f(\mathbf{p} + \epsilon \mathbf{e}_i) - f(\mathbf{p} - \epsilon \mathbf{e}_i)}{2\epsilon} $$
    normalized, where $\epsilon$ is a small offset (e.g., $0.001$) and $\mathbf{e}_i$ are basis vectors.

2.  **Light Setup:** Define the light direction $\mathbf{l}$ and color $\mathbf{c}_L$.
    $$ \mathbf{l} = \text{normalize}(\text{vec3}(-0.5, 0.8, -0.25)) $$
    $$ \mathbf{c}_L = \text{vec3}(0.5, 0.65, 1.0) $$
    [S1, unofficial]

3.  **Diffuse Lighting:**
    $$ d = \max(\mathbf{n} \cdot \mathbf{l}, 0.0) $$
    [S1, unofficial]

4.  **Shadowing:**
    $$ s = \text{softShadow}(\mathbf{p} + \mathbf{n} \cdot 0.03, \mathbf{l}, 12.0) $$
    [S1, unofficial]
    [G] `softShadow` typically performs ray marching along $\mathbf{l}$ with a radius-based softening factor.

5.  **Specular Calculation:**
    Compute the half-vector $\mathbf{h}$:
    $$ \mathbf{h} = \text{normalize}(\mathbf{l} - \mathbf{rd}) $$
    [S1, unofficial]
    Compute specular intensity using a power term interpolated by metalness:
    $$ \text{spec} = \text{pow}(\max(\mathbf{n} \cdot \mathbf{h}, 0.0), \text{mix}(32.0, 140.0, \text{sf.metal})) \cdot s $$
    [S1, unofficial]

6.  **Ambient Occlusion:**
    $$ \text{ao} = \text{calcAO}(\mathbf{p}, \mathbf{n}) $$
    [S1, unofficial]
    [G] `calcAO` typically samples the SDF along $\mathbf{n}$ and neighbors to estimate local occlusion.

7.  **Base Color Accumulation:**
    $$ \mathbf{c}_{\text{base}} = \text{sf.albedo} \cdot (\text{ambient} \cdot \text{ao}) + \text{sf.albedo} \cdot \mathbf{c}_L \cdot d + \mathbf{c}_L \cdot \text{spec} \cdot \text{mix}(0.4, 2.0, \text{sf.metal}) $$
    [S1, unofficial]
    Where `ambient` is a constant $0.16$.

8.  **Fresnel Term:**
    Compute the view angle cosine $\cos \theta_v = \max(\mathbf{n} \cdot (-\mathbf{rd}), 0.0)$.
    $$ F = \text{pow}(1.0 - \cos \theta_v, 5.0) $$
    [S1, unofficial]

9.  **Environment Reflection:**
    Compute the reflection vector $\mathbf{r}$:
    $$ \mathbf{r} = \text{reflect}(\mathbf{rd}, \mathbf{n}) $$
    [G] $\mathbf{r} = 2(\mathbf{n} \cdot (-\mathbf{rd}))\mathbf{n} + \mathbf{rd}$.
    Sample the sky color:
    $$ \mathbf{c}_{\text{refl}} = \text{skyColor}(\mathbf{r}) $$
    [S1, unofficial]

10. **Reflection Contribution:**
    $$ \mathbf{c}_{\text{refl\_contrib}} = \mathbf{c}_{\text{refl}} \cdot \text{mix}(0.03, 0.6, \text{sf.metal}) \cdot (0.25 + 0.75 \cdot F) \cdot \text{ao} $$
    [S1, unofficial]

11. **Final Color:**
    $$ \mathbf{c}_{\text{final}} = \mathbf{c}_{\text{base}} + \mathbf{c}_{\text{refl\_contrib}} + \text{sf.emissive} $$
    [S1, unofficial]

**Parameters:**
- Light Direction: $\text{vec3}(-0.5, 0.8, -0.25)$ [S1, unofficial]
- Light Color: $\text{vec3}(0.5, 0.65, 1.0)$ [S1, unofficial]
- Ambient Intensity: $0.16$ [S1, unofficial]
- Specular Power: Interpolated between $32.0$ (non-metal) and $140.0$ (metal) [S1, unofficial]
- Specular Intensity Multiplier: Interpolated between $0.4$ (non-metal) and $2.0$ (metal) [S1, unofficial]
- Fresnel Power: $5.0$ [S1, unofficial]
- Reflection Intensity: Interpolated between $0.03$ (non-metal) and $0.6$ (metal) [S1, unofficial]
- Fresnel Reflection Weight: $0.25 + 0.75 \cdot F$ [S1, unofficial]

**Edge Cases:**
- If $\mathbf{n}$ is zero (degenerate SDF), the normal must be clamped or defaulted to $\text{vec3}(0,1,0)$ to avoid division by zero in normalization [G].
- If $\mathbf{rd}$ is parallel to $\mathbf{n}$, the half-vector calculation remains stable, but the Fresnel term approaches 0 or 1 depending on alignment.

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for shading, assuming `softShadow` and `calcAO` are $O(k)$ where $k$ is the number of shadow/AO samples. The reflection requires one additional ray trace to the skybox/environment map.
- **Memory Complexity:** $O(1)$ auxiliary memory per pixel.
- **Strengths:** Fast to compute compared to full microfacet BRDFs; provides a convincing "wet" or metallic look with minimal parameters; simple to implement in GLSL/HLSL.
- **Limitations:** The Schlick-like approximation $F = (1-\cos\theta_v)^5$ is less physically accurate than the standard Schlick Fresnel equation for dielectrics; does not account for anisotropy or complex roughness distributions.
- **When to use:** Real-time ray marching for stylized or urban night scenes where performance is critical and approximate metallic reflections are sufficient.

## Background
This model originates from the "neon-raymarcher" project by PIGJET, designed to render a stylized cyberpunk cityscape using SDF ray marching [S1, unofficial]. It adapts the classic Blinn-Phong model to emphasize high-contrast specular highlights and environment reflections typical of neon-lit wet surfaces.

## Sources
[S1] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
