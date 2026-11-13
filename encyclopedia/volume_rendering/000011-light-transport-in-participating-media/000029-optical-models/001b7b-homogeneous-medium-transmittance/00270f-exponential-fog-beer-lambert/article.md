# Exponential Fog (Beer-Lambert)
Also called: Depth Fog, Distance Fog, Classic Fog, Power Fog

## Overview
Exponential fog is a depth-cueing technique that attenuates surface brightness exponentially with view distance, fading colors toward a background (sky) color to simulate atmospheric density or limited visibility [S1, S2, S6, unofficial]. It is based on the Beer-Lambert law, where light intensity decays exponentially as it travels through a participating medium [S5, unofficial]. Unlike linear distance fog, which is a geometric approximation, exponential fog models physical light absorption, providing a more realistic falloff where distant objects dissolve into the horizon [S1, S4, S5, unofficial]. It is commonly used to hide far-clip plane artifacts and tile repetition in large scenes [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `col`: The color of the object hit by the ray (vec3) [S5, unofficial].
    *   `fogColor`: The color of the fog/sky (vec3) [S1, S2, S5, unofficial].
    *   `viewDist` (or `d`): The Euclidean distance from the camera to the hit point (float) [S1, S5, unofficial].
    *   `FOG_DENSITY` (or `α`, `k`): The absorption or attenuation coefficient controlling the rate of fog accumulation (float) [S1, S2, S3, S5, unofficial].
    *   `FOG_START` (optional): The distance from the camera where fog begins to appear (float) [S1, unofficial].
*   **Outputs:**
    *   `finalColor`: The blended color of the object and the fog (vec3) [S1, S2, unofficial].

**Data Structures:**
No complex data structures are required. The method operates on scalar distances and vector colors.

**Parameters and Typical Values:**
*   `FOG_DENSITY`: Controls the rate of fog accumulation. Typical values range from `0.00007` [S3, unofficial] to `0.02` [S2, unofficial]. In some implementations, a coefficient like `1.5` is multiplied by a base density [S4, unofficial].
*   `FOG_START`: The distance at which fog begins. If not specified, fog starts at distance 0 [S1, unofficial].
*   `FOG_EXPONENT`: In some variants, an additional exponent is applied to the distance term to control the curve shape [S4, unofficial].

**Formulas:**
The core transmittance is derived from Beer's Law: $I = I_0 e^{-\alpha d}$ [S5, unofficial]. The fog factor (opacity of the fog) is calculated as:

$$
fogRaw = 1 - e^{-\alpha \cdot \max(0.0, d - d_{start})}
$$

Where:
*   $\alpha$ is `FOG_DENSITY` [S1, S5, unofficial].
*   $d$ is `viewDist` [S1, S5, unofficial].
*   $d_{start}$ is `FOG_START` [S1, unofficial].

The final color is a linear interpolation (mix) between the object color and the fog color:

$$
C_{final} = mix(C_{obj}, C_{fog}, fogFact)
$$

Where `fogFact` is `fogRaw` clamped to a safe range [S1, unofficial].

**Steps:**
1.  Calculate the view distance $d$ from the camera to the intersection point. If no intersection is found, use the maximum render distance limit [S1, unofficial].
2.  Compute the raw fog factor using the exponential decay formula:
    $$
    fogRaw = 1.0 - \exp(-\text{FOG\_DENSITY} \cdot \max(0.0, d - \text{FOG\_START}))
    $$
    [S1, unofficial].
3.  Clamp `fogRaw` to prevent the fog from completely obscuring the scene (e.g., clamp to `[0.0, 0.97]`) [S1, unofficial]. This ensures the background or sky is never fully blocked if the fog density is high.
4.  Determine the `fogColor`. This is often the sky color [S1, unofficial] or a color influenced by lighting (e.g., sun direction) [S5, unofficial].
5.  Blend the object color with the fog color:
    $$
    finalColor = mix(col, fogColor, fogFact)
    $$
    [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exponential_fog(col: Vector3, fog_color: Vector3, view_dist: float, 
                    fog_density: float, fog_start: float = 0.0) -> Vector3:
    """
    Applies exponential fog to a surface color based on Beer-Lambert law.
    
    Args:
        col: The original color of the object.
        fog_color: The color of the fog/sky.
        view_dist: Distance from camera to hit point.
        fog_density: Attenuation coefficient (alpha).
        fog_start: Distance where fog begins.
        
    Returns:
        Blended color.
    """
    # 1. Calculate effective distance
    # If no hit, view_dist might be a limit; otherwise use max(0.0, t) [S1, unofficial]
    effective_dist = max(0.0, view_dist - fog_start)
    
    # 2. Calculate raw fog factor using Beer-Lambert law
    # fogRaw = 1 - exp(-alpha * d) [S1, S5, unofficial]
    fog_raw = 1.0 - exp(-fog_density * effective_dist)
    
    # 3. Clamp fog factor to prevent total occlusion
    # Clamping to 0.97 prevents the scene from becoming fully fog color [S1, unofficial]
    fog_fact = clamp(fog_raw, 0.0, 0.97)
    
    # 4. Blend colors
    # finalColor = mix(col, fogColor, fogFact) [S1, unofficial]
    final_color = mix(col, fog_color, fog_fact)
    
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Complete Occlusion:** If `fogFact` reaches 1.0, the object is completely hidden. Clamping the factor (e.g., to 0.97) is recommended to maintain visibility of the background or sky [S1, unofficial].
*   **Precision:** For very large distances, floating-point precision may affect the exponential calculation. Using `max(0.0, viewDist - FOG_START)` ensures negative distances do not cause invalid exponentials [S1, unofficial].
*   **Linear vs. Exponential:** Do not confuse with linear distance fog, which uses $1 - \frac{d}{range}$ [S4, unofficial]. Exponential fog provides a smoother, more natural falloff.

## Subtopics

### [Fog Blending with Clouds](4123-fog-blending-with-clouds.md)
This method renders a cloud layer by mixing a fixed cloud color with the background (sky) color using distance-based exponential fog, outputting a semi-transparent result. It sets itself apart from siblings like [Height-based Fog with Directional In-scatter](4400-height-based-fog-with-directional-in-scatter.md) by avoiding per-sample fog calculations; instead, it computes a single weighted average depth for the ray passing through the cloud and applies the fog blend only at that specific depth.

### [Height-based Fog with Directional In-scatter](4400-height-based-fog-with-directional-in-scatter.md)
This method simulates aerial perspective by adding a directional in-scatter term to a height-based fog model, making distant objects appear hazy and colored by the sky or sun while maintaining distance perception.

### [Linear Distance Fog](4409-linear-distance-fog.md)
Linear distance fog is a depth-cueing technique that attenuates surface brightness linearly with view distance, fading colors toward black with a brightness floor to simulate atmospheric density or limited visibility. Unlike [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md), which models physical light absorption via exponential decay, linear fog is a geometric approximation implemented with integer or fixed-point arithmetic for performance.

### Exponential Distance Fog
This variant renders a water plane with distance-based exponential fog and sRGB gamma correction to blend the water surface into the scene [S7, unofficial]. It computes fog distance from clip-space depth and applies an exponential falloff to the alpha channel [S7, unofficial].

**Definition:**
Renders a water plane with distance-based exponential fog and sRGB gamma correction to blend the water surface into the scene [S7, unofficial].

**Steps:**
1.  Sample the texture color [S7, unofficial].
2.  Expand gamma (linearize): `tex_col = pow(tex_col, 2.2)` [S7, unofficial].
3.  Compute fog distance from clip-space depth: `fog_dist = gl_FragCoord.z / gl_FragCoord.w` [S7, unofficial].
4.  Calculate alpha using exponential falloff: `alpha = mix(0.5, 0.0, 1.0 - exp(-0.000002 * fog_dist^2))` [S7, unofficial].
5.  Compress gamma (sRGB): `tex_col = pow(tex_col, 1/2.2)` [S7, unofficial].
6.  Output `vec4(tex_col, alpha)` [S7, unofficial].

**Formula:**
$$
\alpha = \text{mix}(0.5, 0.0, 1.0 - e^{-k \cdot d^2})
$$
Where $d$ is fog distance and $k = 0.000002$ [S7, unofficial].

**Parameters:**
*   `gamma = vec3(2.2)`: Used for linear-to-sRGB conversion [S7, unofficial].
*   `inv_gamma = vec3(1/2.2)`: Used for sRGB-to-linear conversion [S7, unofficial].

**Code:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
// Exponential Distance Fog Variant [S7, unofficial]
uniform sampler2D u_texture_0;
uniform mat4 m_view_proj;

void main() {
    vec2 uv = ...; // Derived from vertex attributes
    vec3 tex_col = texture(u_texture_0, uv).rgb;
    
    // Linearize gamma
    vec3 gamma = vec3(2.2);
    tex_col = pow(tex_col, gamma);
    
    // Compute fog distance from clip-space depth
    // Note: gl_FragCoord.z/w is standard for perspective depth
    float fog_dist = gl_FragCoord.z / gl_FragCoord.w;
    
    // Calculate alpha using exponential falloff
    float k = 0.000002;
    float alpha = mix(0.5, 0.0, 1.0 - exp(-k * fog_dist * fog_dist));
    
    // Compress gamma back to sRGB
    vec3 inv_gamma = vec3(1.0/2.2);
    tex_col = pow(tex_col, inv_gamma);
    
    fragColor = vec4(tex_col, alpha);
}
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel. The calculation involves a few arithmetic operations and a single exponential function call, making it extremely fast compared to volumetric ray marching.
*   **Memory Complexity:** $O(1)$. No additional buffers or textures are required beyond the standard color and depth buffers.
*   **Performance:** This method is highly efficient and can be applied as a post-processing pass or in the fragment shader of opaque geometry [S6, unofficial]. It is suitable for real-time rendering on mobile and desktop hardware.
*   **Strengths:**
    *   Provides realistic atmospheric perspective with minimal computational cost [S5, unofficial].
    *   Effectively hides draw distance limits and far-clip artifacts [S3, unofficial].
    *   Simple to implement and tune [S6, unofficial].
*   **Limitations:**
    *   Does not simulate light scattering or in-scattering within the volume; it only attenuates the background color [S5, unofficial].
    *   Cannot represent volumetric clouds or dense fog layers that obscure geometry from multiple angles (requires ray marching for that) [S6, unofficial].
    *   The exponential falloff may appear too abrupt or too slow depending on the density parameter, requiring careful tuning for specific scene scales [S2, S3, unofficial].
*   **When to Use:** Ideal for outdoor scenes with large visible distances, skyboxes, and terrain rendering where performance is critical and full volumetric simulation is unnecessary [S6, unofficial].
*   **When Not to Use:** When realistic volumetric effects like god rays, thick fog banks, or light scattering are required [S6, unofficial].

## Background
Exponential fog is based on the Beer-Lambert law, which describes the attenuation of light as it passes through a participating medium [S1, S5, unofficial]. The law states that the intensity of light decreases exponentially with the distance traveled and the density of the medium [S5, unofficial]. This physical model has been widely adopted in computer graphics to simulate atmospheric effects. Inigo Quilez popularized a specific implementation of this fog in his raymarching tutorials, which includes height-based falloff and sun-colored fog [S5, unofficial]. The technique is a standard feature in many game engines and rendering pipelines, often referred to as "distance fog" or "exponential height fog" [S6, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S3] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S4] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
[S5] Painting with Math: A Gentle Study of Raymarching, https://blog.maximeheckel.com/posts/painting-with-math-a-gentle-study-of-raymarching/, fetched 2026-10-27 (unofficial)
[S6] Building beautiful worlds across platforms with PlayCanvas and the Magnopus Web Renderer; featuring volumetric lighting — Mag..., https://www.magnopus.com/blog/building-beautiful-worlds-across-platforms-with-playcanvas-and-the-magnopus-web-renderer-featuring-volumetric-lighting, fetched 2026-10-29 (unofficial)
[S7] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
