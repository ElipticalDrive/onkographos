# SDF Phong Lighting

Also called: SDF Directional Lighting, TextMeshPro Lighting

## Overview
SDF Phong Lighting is a shading technique that applies directional diffuse and specular lighting to Signed Distance Field (SDF) volumes, such as text or meshes, by estimating surface normals from the SDF gradient [S1, unofficial]. It solves the problem of flat, unlit volume rendering by providing geometric cues that allow standard Phong reflection model terms to interact with the volume's internal structure [S1, unofficial]. Unlike parent methods that may rely on pre-computed or analytic normals, this approach computes normals on-the-fly using finite differences, making it applicable to any discrete scalar field without additional preprocessing [S1, unofficial]. It is distinct from specular or environment-based shading models by focusing on the core Phong components (diffuse and specular) driven by a single configurable light direction [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A signed distance value $d$ at a point $p$, the spatial gradient of the SDF $\nabla d$ (approximated via finite differences), and a set of lighting parameters.
- **Output:** The final fragment color $C_{final}$.

**Data Structures:**
- **Lighting Parameters:** A configuration struct containing:
  - `_LightAngle` (float): Direction of the light in radians (default 3.1416).
  - `_SpecularColor` (vec3): Color of the specular highlight (default 1,1,1).
  - `_SpecularPower` (float): Intensity multiplier for specular (default 2.0).
  - `_Reflectivity` (float): Shininess exponent (default 10).
  - `_Diffuse` (float): Diffuse lighting factor (default 0.5).
  - `_Ambient` (float): Ambient lighting factor (default 0.5).

**Steps:**
1. **Compute Light Direction:** Derive the normalized light direction vector $\mathbf{l}$ from `_LightAngle`. In 2D SDF text rendering, this is often a fixed direction in the XY plane. In 3D, it may be a uniform direction vector.
   $$ \mathbf{l} = (\cos(\text{\_LightAngle}), \sin(\text{\_LightAngle}), 0) $$
   [G: Normalization is assumed if not explicitly unit-length in source, though angle-based derivation implies unit length in 2D projection].

2. **Estimate Normal:** Obtain the surface normal $\mathbf{n}$ by normalizing the SDF gradient $\nabla d$.
   $$ \mathbf{n} = \frac{\nabla d}{\|\nabla d\|} $$

3. **Compute Specular Term:** Calculate the specular contribution using the Phong specular model.
   $$ \text{spec} = \text{pow}(\max(0, \mathbf{n} \cdot \mathbf{l}), \text{\_Reflectivity}) $$
   Scale by specular color and power:
   $$ C_{spec} = \text{\_SpecularColor} \times \text{spec} \times \text{\_SpecularPower} $$
   [S1, unofficial]

4. **Compute Diffuse Term:** Calculate the diffuse attenuation. The source formula uses a specific variation:
   $$ C_{diff} = 1 - (\mathbf{n} \cdot \mathbf{l}) \times \text{\_Diffuse} $$
   [S1, unofficial]
   Note: This differs from the standard Lambertian $1 - \mathbf{n} \cdot \mathbf{l}$ or $\max(0, \mathbf{n} \cdot \mathbf{l})$ by using a subtraction model scaled by a factor, effectively darkening the surface based on the dot product.

5. **Compute Ambient Term:** Apply an ambient term that varies based on the normal's Z-component (assuming a top-down light or view direction alignment in 2D text):
   $$ C_{amb} = \text{lerp}(\text{\_Ambient}, 1, \mathbf{n}_z^2) $$
   [S1, unofficial]

6. **Combine:** Multiply the base color (or face color) by the diffuse and ambient terms, then add the specular term.
   $$ C_{final} = C_{base} \times C_{diff} \times C_{amb} + C_{spec} $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_phong_lighting(n: Vec3, base_color: Vec3, params: LightingParams) -> Vec3:
    # 1. Compute light direction from angle
    # Assuming 2D light direction in XY plane for text SDFs
    l = Vec3(cos(params._LightAngle), sin(params._LightAngle), 0.0)
    
    # 2. Specular calculation
    # n and l are assumed normalized
    dot_nl = max(0.0, dot(n, l))
    spec = pow(dot_nl, params._Reflectivity)
    specular_color = params._SpecularColor * spec * params._SpecularPower
    
    # 3. Diffuse calculation
    # Formula from source: 1 - (n . l) * Diffuse
    # Note: This can result in negative values if dot_nl > 1/Diffuse, 
    # but Diffuse is typically <= 1.0. Source does not clamp.
    diffuse_factor = 1.0 - (dot_nl * params._Diffuse)
    
    # 4. Ambient calculation
    # lerp(Ambient, 1, n.z^2)
    ambient_factor = lerp(params._Ambient, 1.0, n.z * n.z)
    
    # 5. Combine
    # base_color is the SDF face color (e.g., text color)
    final_color = base_color * diffuse_factor * ambient_factor + specular_color
    
    return final_color
```

**Edge Cases and Pitfalls:**
- **Normal Degeneracy:** If $\nabla d$ is zero (e.g., at the exact center of a sphere or far from the surface), normalization fails. A guard clause or fallback normal is required [G].
- **Negative Diffuse:** The formula $1 - (\mathbf{n} \cdot \mathbf{l}) \times \text{\_Diffuse}$ can produce negative lighting if $\mathbf{n} \cdot \mathbf{l}$ is large and $\text{\_Diffuse} > 1$. Standard practice is to clamp the final color or the diffuse factor [G].
- **Light Angle Interpretation:** The source uses `_LightAngle` as a single float, implying a 2D direction. In 3D volumes, this parameter is insufficient without an elevation angle or a full vector input [S1, unofficial].

## Subtopics

### SDF Glow Effect
Adds a colored glow around the text by computing a falloff function based on the signed distance and applying it to a glow color [S1, unofficial]. It is chosen when a neon or emissive halo effect is desired around the SDF geometry [S1, unofficial]. Parameters include `_GlowColor` (default 0,1,0,0.5), `_GlowOffset` (default 0), `_GlowInner` (default 0.05), `_GlowOuter` (default 0.05), and `_GlowPower` (default 0.75) [S1, unofficial]. The implementation calls `GetGlowColor(sd, scale)` and adds the result to the face color: `faceColor.rgb += glowColor.rgb * glowColor.a` [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** $O(1)$ per fragment. The lighting calculation involves a few trigonometric functions, dot products, and power operations, all constant time relative to scene size.
- **Memory:** $O(1)$ auxiliary memory. Only lighting parameters and intermediate vectors are stored.

**Measured Performance:**
- Performance is hardware-dependent but generally fast on modern GPUs due to the simplicity of the math. The use of `pow` for specular can be slightly expensive but is negligible for text rendering.

**Strengths and Limitations:**
- **Strengths:** Simple to implement; provides immediate visual depth to SDF text; configurable via simple scalar parameters.
- **Limitations:** The diffuse model is non-standard (subtraction-based) and may not behave physically plausibly; the light direction is limited to a 2D angle in the source implementation, restricting 3D applicability without extension; no shadowing is included.

**When to Use:**
- Use for 2D text rendering or simple 3D volumes where a single directional light is sufficient and performance is critical.

**When Not to Use:**
- Do not use for physically based rendering (PBR) or scenes requiring multiple lights, shadows, or complex material interactions.

## Background
Originating from the Unity NanoVDB renderer project by Anders Blomqvist, this method is an unofficial implementation of Phong shading tailored for SDF text rendering [S1, unofficial]. It draws inspiration from TextMeshPro's lighting system, adapting the Phong model to work with gradient-based normals derived from the SDF [S1, unofficial]. It is primarily used in real-time UI and text rendering within Unity environments.

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
