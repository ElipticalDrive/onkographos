# Terrain Shader Lighting and Effects


## Overview
Terrain Shader Lighting and Effects is a GLSL 330 fragment shader technique for chunk-based voxel terrain that combines baked vertex colors with dynamic lighting from a directional sun, spherical occluders, and local point lights [S1, unofficial]. It operates on pre-rasterized triangle meshes to compute lighting and screen-space effects like grid overlays and mode-switch visual rings [S1, unofficial].

## How it works

### Inputs and Outputs
**Vertex Shader Inputs:**
*   `vertexPosition`: `vec3`
*   `vertexNormal`: `vec3`
*   `vertexColor`: `vec4` (RGB: albedo, A: self-lit mix factor) [S1, unofficial]

**Uniforms:**
*   `mvp`: `mat4` (Model-View-Projection matrix) [S1, unofficial]
*   `matModel`: `mat4` (Model matrix) [S1, unofficial]
*   `sunDirection`: `vec3` [S1, unofficial]
*   `sunColor`: `vec3` [S1, unofficial]
*   `ambientColor`: `vec3` [S1, unofficial]
*   `cameraPosition`: `vec3` [S1, unofficial]
*   `fogColor`: `vec3` [S1, unofficial]
*   `fogStart`: `float` [S1, unofficial]
*   `fogEnd`: `float` [S1, unofficial]
*   `occluderCount`: `int` [S1, unofficial]
*   `occluders`: `vec4[MAX_OCCLUDERS]` (xyz: center, w: radius) [S1, unofficial]
*   `pointLightCount`: `int` [S1, unofficial]
*   `pointLights`: `vec4[MAX_POINT_LIGHTS]` (xyz: position, w: range) [S1, unofficial]
*   `pointLightColors`: `vec4[MAX_POINT_LIGHTS]` [S1, unofficial]
*   `waveCenter`: `vec3` [S1, unofficial]
*   `waveRadius`: `float` [S1, unofficial]
*   `waveWidth`: `float` [S1, unofficial]
*   `waveStrength`: `float` [S1, unofficial]
*   `waveColor`: `vec3` [S1, unofficial]
*   `gridSpacing`: `float` [S1, unofficial]
*   `gridStrength`: `float` [S1, unofficial]
*   `worldShade`: `float` [S1, unofficial]

**Constants:**
*   `MAX_OCCLUDERS = 8` [S1, unofficial]
*   `MAX_POINT_LIGHTS = 8` [S1, unofficial]

**Fragment Shader Output:**
*   `fragColor`: `vec4` (Final color with fog) [S1, unofficial]

### Data Structures
Occluders are packed into a single `vec4` array where the first three components represent the 3D center coordinates and the fourth component represents the radius [S1, unofficial]. Point lights are similarly packed with position in `xyz` and attenuation range in `w` [S1, unofficial].

### Steps

1.  **Vertex Transformation:**
    The vertex shader transforms the normal to world space using the model matrix:
    $$ \mathbf{n}_{world} = \text{mat3}(\text{matModel}) \cdot \mathbf{n}_{vertex} $$
    It passes the vertex color and world position to the fragment shader:
    $$ \mathbf{p}_{world} = \text{matModel} \cdot \begin{bmatrix} \mathbf{p}_{vertex} \\ 1 \end{bmatrix} $$
    The clip position is computed as:
    $$ \mathbf{p}_{clip} = \text{mvp} \cdot \begin{bmatrix} \mathbf{p}_{vertex} \\ 1 \end{bmatrix} $$
    [S1, unofficial]

2.  **Sun Visibility Calculation:**
    For each occluder, the shader determines if it blocks the sun [S1, unofficial].
    *   Compute vector from surface to occluder center: $\mathbf{r} = \mathbf{c}_{occluder} - \mathbf{p}_{world}$ [S1, unofficial].
    *   Compute projection of $\mathbf{r}$ onto the sun direction $\mathbf{l}_{sun} = -\mathbf{d}_{sun}$:
        $$ d_{along} = \mathbf{r} \cdot \mathbf{l}_{sun} $$
    *   If $d_{along} \le 0$, the occluder is behind the surface; skip [S1, unofficial].
    *   Compute perpendicular distance:
        $$ \mathbf{r}_{perp} = \mathbf{r} - d_{along} \mathbf{l}_{sun} $$
        $$ d_{perp} = \| \mathbf{r}_{perp} \| $$
    *   Calculate visibility using a smoothstep based on the occluder radius $R$:
        $$ v_{sun} = \text{smoothstep}(0.8 R, 1.2 R, d_{perp}) $$
    *   Combine visibility factors multiplicatively across all occluders: $V_{sun} = \prod v_{sun}$ [G].

3.  **Directional Face Shading:**
    Compute a directional shade factor based on the normal $\mathbf{n}$:
    $$ \text{shade}_{dir} = n_x^2 \cdot 0.80 + n_z^2 \cdot 0.72 + \begin{cases} n_y^2 \cdot 1.00 & \text{if } n_y > 0 \\ n_y^2 \cdot 0.55 & \text{if } n_y \le 0 \end{cases} $$
    This blends face shades based on squared normal components, making top faces brighter and bottom faces darker [S1, unofficial].

4.  **Point Light Attenuation:**
    For each point light:
    *   Compute distance $d = \| \mathbf{p}_{world} - \mathbf{p}_{light} \|$ [S1, unofficial].
    *   If $d \ge \text{range}$, skip [S1, unofficial].
    *   Compute attenuation:
        $$ A = \left( 1.0 - \frac{d}{\text{range}} \right)^2 $$
    *   Compute Lambertian term:
        $$ L = \max( \mathbf{n} \cdot \frac{\mathbf{p}_{light} - \mathbf{p}_{world}}{\max(d, 0.0001)}, 0.0 ) $$
    *   Add to sum: $\mathbf{C}_{points} += \mathbf{C}_{light} \cdot L \cdot A$ [S1, unofficial].

5.  **Final Color Composition:**
    *   Normalize normal $\mathbf{n}$ [S1, unofficial].
    *   Compute albedo:
        $$ \mathbf{C}_{albedo} = \mathbf{C}_{vertex} \cdot \text{mix}(1.0, \text{shade}_{dir}, \text{worldShade}) $$
    *   Compute diffuse sunlight:
        $$ \mathbf{C}_{sun} = \mathbf{C}_{sun\_color} \cdot \max( \mathbf{n} \cdot (-\mathbf{d}_{sun}), 0.0 ) \cdot V_{sun} $$
    *   Combine:
        $$ \mathbf{C}_{lit} = \mathbf{C}_{albedo} \cdot (\mathbf{C}_{ambient} + \mathbf{C}_{sun} + \mathbf{C}_{points}) $$
    *   Mix between lit and self-lit (vertex alpha):
        $$ \mathbf{C}_{final} = \text{mix}(\mathbf{C}_{lit}, \mathbf{C}_{vertex}, \mathbf{C}_{vertex}.a) $$
    [S1, unofficial]

6.  **Grid Overlay:**
    If `gridStrength > 0.0`:
    *   Compute grid coordinates: $\mathbf{p}_{grid} = \mathbf{p}_{world} / \text{gridSpacing}$ [S1, unofficial].
    *   Compute width: $w = \max(\text{fwidth}(\mathbf{p}_{grid}), 10^{-4})$ [S1, unofficial].
    *   Compute distance to grid lines:
        $$ \mathbf{g} = \frac{| \text{fract}(\mathbf{p}_{grid} - 0.5) - 0.5 |}{w} + \| \mathbf{n} \| \cdot 10.0 $$
    *   Compute line mask:
        $$ \text{line} = \text{clamp}(1.0 - \min(\min(g_x, g_y), g_z), 0.0, 1.0) $$
    *   Fade with distance:
        $$ \text{near} = 1.0 - \text{smoothstep}(6.0, 14.0, \| \mathbf{p}_{world} - \mathbf{p}_{camera} \|) $$
    *   Apply:
        $$ \mathbf{C}_{final} *= 1.0 - 0.16 \cdot \text{line} \cdot \text{near} \cdot \text{gridStrength} $$
    [S1, unofficial]

7.  **Mode-Switch Wave:**
    If `waveStrength > 0.0`:
    *   Compute distance to wave center in XZ plane: $d = \| \mathbf{p}_{world}.xz - \text{waveCenter}.xz \|$ [S1, unofficial].
    *   Compute band:
        $$ \text{band} = 1.0 - \text{smoothstep}(0.0, \text{waveWidth}, | d - \text{waveRadius} |) $$
    *   Compute rim lighting:
        $$ \mathbf{v}_{toEye} = \mathbf{p}_{camera} - \mathbf{p}_{world} $$
        $$ \text{rim} = \text{pow}(1.0 - \max( \mathbf{n} \cdot \frac{\mathbf{v}_{toEye}}{\| \mathbf{v}_{toEye} \|}, 0.0 ), 2.0) $$
    *   Add wave color:
        $$ \mathbf{C}_{wave} = \mathbf{C}_{wave\_color} \cdot \text{band} \cdot \text{waveStrength} \cdot (0.3 + 0.5 \cdot \text{rim}) $$
    *   Mute outside terrain by mixing with luminance:
        $$ \text{lum} = \mathbf{C}_{final} \cdot \begin{bmatrix} 0.299 \\ 0.587 \\ 0.114 \end{bmatrix} $$
        $$ \mathbf{C}_{final} = \text{mix}(\mathbf{C}_{final}, \text{lum}, \text{smoothstep}(\text{waveRadius}, \text{waveRadius} + \text{waveWidth}, d)) $$
    [S1, unofficial]

8.  **Fog:**
    *   Compute eye distance: $d_{eye} = \| \mathbf{p}_{world} - \mathbf{p}_{camera} \|$ [S1, unofficial].
    *   Compute fog factor:
        $$ f = \text{smoothstep}(\text{fogStart}, \text{fogEnd}, d_{eye}) $$
    *   Mix with fog color:
        $$ \mathbf{C}_{final} = \text{mix}(\mathbf{C}_{final}, \mathbf{C}_{fog}, f) $$
    [S1, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def terrain_shader_lighting(vertex_position, vertex_normal, vertex_color, uniforms):
    """
    Computes the final color for a terrain fragment.
    
    Args:
        vertex_position: vec3
        vertex_normal: vec3
        vertex_color: vec4 (RGB albedo, A self-lit mix)
        uniforms: dict containing all uniform values
        
    Returns:
        vec4: Final color
    """
    # Helper: GLSL intrinsic fwidth(p) = abs(dFdx(p)) + abs(dFdy(p))
    def fwidth(p):
        return abs(p.x - p.x_prev) + abs(p.y - p.y_prev)

    # 1. Vertex Transform (world space)
    frag_position = uniforms['matModel'] @ vec4(vertex_position, 1.0)[:3]
    frag_normal = normalize(uniforms['matModel'][:3, :3] @ vertex_normal)
    
    # 2. Sun Visibility
    sun_visibility = 1.0
    for i in range(uniforms['occluderCount']):
        occluder = uniforms['occluders'][i]
        center = occluder[:3]
        radius = occluder[3]
        
        to_occluder = center - frag_position
        to_sun = -uniforms['sunDirection']
        
        along = dot(to_occluder, to_sun)
        if along <= 0.0:
            continue
            
        perp_vec = to_occluder - to_sun * along
        perp_dist = length(perp_vec)
        
        vis = smoothstep(radius * 0.80, radius * 1.20, perp_dist)
        sun_visibility *= vis
        
    # 3. Directional Shade
    n = frag_normal
    shade_dir = n.x**2 * 0.80 + n.z**2 * 0.72 + (n.y**2 * 1.00 if n.y > 0.0 else n.y**2 * 0.55)
    
    # 4. Point Lights
    point_light_sum = vec3(0.0)
    for i in range(uniforms['pointLightCount']):
        light = uniforms['pointLights'][i]
        light_color = uniforms['pointLightColors'][i][:3]
        range_val = light[3]
        
        to_light = light[:3] - frag_position
        dist = length(to_light)
        
        if dist >= range_val:
            continue
            
        attenuation = (1.0 - dist / range_val)**2
        normal_to_light = normalize(to_light)
        lambert = max(dot(n, normal_to_light), 0.0)
        
        point_light_sum += light_color * lambert * attenuation
        
    # 5. Final Color Composition
    albedo = vertex_color[:3] * mix(1.0, shade_dir, uniforms['worldShade'])
    
    sun_diffuse = max(dot(n, -uniforms['sunDirection']), 0.0) * uniforms['sunColor'] * sun_visibility
    ambient = uniforms['ambientColor']
    
    lit_color = albedo * (ambient + sun_diffuse + point_light_sum)
    
    final_color = mix(lit_color, vertex_color[:3], vertex_color[3])
    
    # 6. Grid Overlay
    if uniforms['gridStrength'] > 0.0:
        p_grid = frag_position / uniforms['gridSpacing']
        w = max(fwidth(p_grid), 1e-4)
        g = abs(fract(p_grid - 0.5) - 0.5) / w + length(n) * 10.0
        line = clamp(1.0 - min(min(g.x, g.y), g.z), 0.0, 1.0)
        eye_dist = length(frag_position - uniforms['cameraPosition'])
        near = 1.0 - smoothstep(6.0, 14.0, eye_dist)
        final_color *= 1.0 - 0.16 * line * near * uniforms['gridStrength']
        
    # 7. Mode-Switch Wave
    if uniforms['waveStrength'] > 0.0:
        d = length(frag_position.xz - uniforms['waveCenter'].xz)
        band = 1.0 - smoothstep(0.0, uniforms['waveWidth'], abs(d - uniforms['waveRadius']))
        
        to_eye = uniforms['cameraPosition'] - frag_position
        rim = pow(1.0 - max(dot(n, normalize(to_eye)), 0.0), 2.0)
        
        wave_color = uniforms['waveColor'] * band * uniforms['waveStrength'] * (0.3 + 0.5 * rim)
        
        lum = dot(final_color, vec3(0.299, 0.587, 0.114))
        mute_factor = smoothstep(uniforms['waveRadius'], uniforms['waveRadius'] + uniforms['waveWidth'], d)
        final_color = mix(final_color, vec3(lum), mute_factor)
        
        final_color += wave_color
        
    # 8. Fog
    eye_dist = length(frag_position - uniforms['cameraPosition'])
    fog = smoothstep(uniforms['fogStart'], uniforms['fogEnd'], eye_dist)
    final_color = mix(final_color, uniforms['fogColor'], fog)
    
    return vec4(final_color, 1.0)
```

### Parameters
*   `MAX_OCCLUDERS`: 8 [S1, unofficial]
*   `MAX_POINT_LIGHTS`: 8 [S1, unofficial]
*   `gridSpacing`: Not given in the sources.
*   `fogStart`, `fogEnd`: Not given in the sources.
*   `waveRadius`, `waveWidth`: Not given in the sources.

### Edge Cases and Pitfalls
*   **Precision:** The `fwidth` calculation for the grid overlay requires careful handling of precision to avoid artifacts at large distances. The source uses `max(fwidth(p), 1e-4)` to prevent division by zero [S1, unofficial].
*   **Boundaries:** The point light attenuation formula `(1.0 - distance / range)^2` ensures zero contribution at the range limit, avoiding discontinuities [S1, unofficial].
*   **Degenerate Input:** If the normal is zero, the shading calculations will fail. The source assumes valid normals [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** $O(N_{occluders} + N_{lights})$ per fragment, where $N_{occluders}$ is the number of spherical occluders and $N_{lights}$ is the number of point lights. The grid and wave effects are $O(1)$ [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per fragment, as it only requires uniforms and vertex attributes [S1, unofficial].

### Measured Performance
*   Not given in the sources.

### Scaling
*   The method scales linearly with the number of occluders and point lights. The number of occluders and lights is capped at 8 each [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Efficient for chunk-based terrain; allows for complex visual effects (grid, wave) alongside lighting; uses baked vertex colors for ambient occlusion [S1, unofficial].
*   **Limitations:** Limited to pre-rasterized meshes; does not handle dynamic volumetric shadows; limited number of lights and occluders [S1, unofficial].

### When to Use
*   Use for chunk-based voxel terrain where performance is critical and dynamic lighting is limited to a few sources [S1, unofficial].
*   Do not use for fully volumetric scenes or when dynamic, per-voxel lighting is required [S1, unofficial].

## Background

### Origin
The method is described in the context of the "Terraformer" project, a voxel engine/game launcher [S1, unofficial].

### History
The method is part of the "Terraformer" project, which is a game engine for voxel terrain [S1, unofficial].

### Usage
Used in the "Terraformer" project for rendering chunk-based voxel terrain [S1, unofficial].

## Sources

[S1] GhostlyActive/VoxelTerraformer, https://github.com/GhostlyActive/VoxelTerraformer, fetched 2026-10-27 (unofficial)
