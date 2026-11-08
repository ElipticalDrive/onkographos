# Volume Ray Marching with Temporal Accumulation

Also called: HZL Fluid Effects Volume Rendering

## Overview
Volume Ray Marching with Temporal Accumulation is a real-time rendering technique that marches rays through a 3D density field to compute volumetric lighting and opacity, stabilized over time by blending the current frame's result with a history texture [S1, unofficial]. It distinguishes itself from standard ray marching by incorporating temporal reprojection and jittering to reduce noise and aliasing artifacts, making it suitable for dynamic fluid simulations where static sampling would produce flickering [S1, unofficial].

## How it works
The method operates in a fragment shader, processing pixels in screen space. It constructs view rays, intersects them with a canonical volume box and scene geometry, and marches through a density texture. The core innovation is the accumulation of results over multiple frames using a history buffer, weighted by a temporal factor.

### Inputs and Outputs
**Inputs:**
- Camera matrices: `viewMatrix`, `projectionMatrix` [S1, unofficial].
- Camera position: `cameraPosition` [S1, unofficial].
- Texture coordinates: `textureCoordinate` (screen space, 0.0 to 1.0) [S1, unofficial].
- Density field: `densityTexture` (3D texture, sampled at `position * 0.5 + 0.5`) [S1, unofficial].
- Scene depth: `sceneDepthTexture` (2D depth buffer) [S1, unofficial].
- History buffer: `historyTexture` (2D RGBA texture, previous frame's accumulation) [S1, unofficial].
- Uniforms: `lightDirection`, `lightIntensity`, `stepSize`, `maximumSteps`, `densityScale`, `absorption`, `emissionStrength`, `fieldMode`, `temporalEnabled`, `historyWeight`, `frameIndex`, `jitterEnabled`, `selfShadowingEnabled` [S1, unofficial].

**Outputs:**
- Final color: `outColor` (RGBA) [S1, unofficial].

### Data Structures
The implementation relies on OpenGL textures with specific formats and filtering:
- `m_colorTexture`: `GL_RGBA8`, filtering `GL_LINEAR`, wrap `GL_CLAMP_TO_EDGE` [S1, unofficial].
- `m_depthTexture`: `GL_DEPTH_COMPONENT24`, filtering `GL_NEAREST`, wrap `GL_CLAMP_TO_EDGE` [S1, unofficial].
- `m_sceneDepthTexture`: `GL_DEPTH_COMPONENT24`, filtering `GL_NEAREST`, wrap `GL_CLAMP_TO_EDGE` [S1, unofficial].
- `m_historyTexture`: `GL_RGBA8`, filtering `GL_LINEAR`, wrap `GL_CLAMP_TO_EDGE` [S1, unofficial]. Cleared to zero at initialization [S1, unofficial].

### Algorithm Steps

1.  **Ray Setup**:
    Convert the fragment's texture coordinate `uv` to clip space:
    $$ \mathbf{clipPos} = \text{vec4}(uv \cdot 2.0 - 1.0, 1.0, 1.0) $$
    Unproject the far plane point using the inverse view-projection matrix $M_{vp}^{-1}$:
    $$ \mathbf{farWorld} = \frac{\text{vec3}(M_{vp}^{-1} \cdot \mathbf{clipPos})}{\mathbf{clipPos}.w} $$
    Compute the ray direction:
    $$ \mathbf{d} = \text{normalize}(\mathbf{farWorld} - \mathbf{o}) $$
    where $\mathbf{o}$ is `cameraPosition` [S1, unofficial].

2.  **Box Intersection**:
    Intersect the ray $(\mathbf{o}, \mathbf{d})$ with the canonical box $[-1, 1]^3$. Compute inverse direction $\mathbf{i_d} = 1.0 / \mathbf{d}$.
    $$ \mathbf{t_1} = (-1.0 - \mathbf{o}) \cdot \mathbf{i_d} $$
    $$ \mathbf{t_2} = (1.0 - \mathbf{o}) \cdot \mathbf{i_d} $$
    $$ t_{min} = \max(\min(\mathbf{t_1}), \min(\mathbf{t_2})) $$
    $$ t_{max} = \min(\max(\mathbf{t_1}), \max(\mathbf{t_2})) $$
    The intersection interval is $[t_{min}, t_{max}]$ [S1, unofficial].

3.  **Scene Clipping**:
    Sample `sceneDepthTexture` at `uv` to get `sceneDepth`. If `sceneDepth < 1.0`, unproject the scene point to world space to find `sceneDistance` along $\mathbf{d}$. Update the exit distance:
    $$ t_{max} = \min(t_{max}, \text{sceneDistance}) $$
    If $t_{max} \le t_{min}$, discard the fragment (no volume intersection) [S1, unofficial].

4.  **Jittering**:
    If `jitterEnabled`, compute a random offset $j$ using `frameIndex` and `gl_FragCoord`.
    $$ j = \text{random}(gl\_FragCoord.xy, \text{frameIndex}) \cdot \text{stepSize} $$
    $$ t_{start} = \max(t_{min}, 0.0) + j $$
    The random function is defined as:
    $$ \mathbf{v} = \text{fract}(\text{vec3}(coord.xyx) \cdot 0.1031) $$
    $$ \mathbf{v} += \text{dot}(\mathbf{v}, \mathbf{v}.yzx + 33.33 + \text{frameIndex} \cdot 0.01) $$
    $$ j = \text{fract}((\mathbf{v}.x + \mathbf{v}.y) \cdot \mathbf{v}.z) $$
    [S1, unofficial].

5.  **Initialization**:
    Initialize the accumulated color and opacity:
    $$ \mathbf{accumulated} = \text{vec4}(0.0) $$
    Initialize flags: `foundVolume = false`, `representativePosition = vec3(0.0)` [S1, unofficial].

6.  **Ray Marching Loop**:
    Iterate $i$ from 0 to `maximumSteps` (typically 767). Break if $i \ge \text{maximumSteps}$ or $t_{current} \ge t_{max}$.
    $$ \mathbf{p} = \mathbf{o} + \mathbf{d} \cdot t_{current} $$
    Sample density:
    $$ \rho = \text{texture}(\text{densityTexture}, \mathbf{p} \cdot 0.5 + 0.5).r $$
    
    **Volume Entry Detection**:
    If `!foundVolume` and $\rho > 0.01$:
    $$ \text{representativePosition} = \mathbf{p} $$
    $$ \text{foundVolume} = \text{true} $$
    [S1, unofficial].

    **Color Computation**:
    Compute `smokeColor` and `emittedLight` based on `fieldMode`:
    - **Mode 0 (Smoke/Fire)**: Sample temperature and fire textures. Compute light transmittance (see Step 7). Compute ambient light $\mathbf{L}_{amb} = (0.12, 0.15, 0.20)$. Compute direct light $\mathbf{L}_{dir} = (1.0, 0.88, 0.72) \cdot \text{lightIntensity} \cdot \text{transmittance}$. Mix smoke color based on $\mathbf{p}.y$. Compute flame color from temperature.
      $$ \mathbf{E} = \text{flameColor} \cdot \text{fire} \cdot \text{emissionStrength} $$
    - **Mode 1 (Simple Smoke)**:
      $$ \text{smokeColor} = \text{mix}((0.3, 0.04, 0.01), (1.0, 0.65, 0.08), \rho) $$
    - **Mode 2/3 (Abstract)**:
      $$ \rho' = |\rho - 0.5| \cdot 2.0 $$
      $$ \text{smokeColor} = \begin{cases} (1.0, 0.0, 0.0) & \rho \ge 0.5 \\ (0.0, 0.0, 1.0) & \rho < 0.5 \end{cases} $$
    [S1, unofficial].

    **Accumulation**:
    Compute opacity for the step:
    $$ \alpha = 1.0 - e^{-\rho \cdot \text{stepSize}} $$
    Update accumulated color and opacity using alpha blending:
    $$ \mathbf{accumulated}.rgb += \text{smokeColor} \cdot \alpha \cdot (1.0 - \mathbf{accumulated}.a) $$
    $$ \mathbf{accumulated}.a += \alpha \cdot (1.0 - \mathbf{accumulated}.a) $$
    [S1, unofficial].

    Advance $t_{current}$ by `stepSize` [S1, unofficial].

7.  **Light Transmittance (Self-Shadowing)**:
    If `selfShadowingEnabled`, march 20 steps in direction $\mathbf{l} = \text{normalize}(\text{lightDirection} + (0.0001, 0.0001, 0.0001))$ with step size 0.08.
    $$ \text{opticalDepth} = 0.0 $$
    For each step:
    $$ \mathbf{p}_s = \mathbf{p} + \mathbf{l} \cdot t_s $$
    $$ \text{opticalDepth} += \text{texture}(\text{densityTexture}, \mathbf{p}_s \cdot 0.5 + 0.5).r \cdot 0.08 $$
    Break if $|\mathbf{p}_s| > 1.0$.
    $$ \text{transmittance} = e^{-\text{opticalDepth} \cdot \text{absorption} \cdot \text{densityScale} \cdot 3.0} $$
    [S1, unofficial].

8.  **Temporal Accumulation**:
    If `temporalEnabled`:
    $$ \mathbf{history} = \text{texture}(\text{historyTexture}, uv).rgb $$
    $$ \mathbf{outColor} = \text{mix}(\mathbf{history}, \mathbf{accumulated}.rgb, \text{historyWeight}) $$
    Else:
    $$ \mathbf{outColor} = \mathbf{accumulated}.rgb $$
    [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_ray_marching_with_temporal_accumulation(uv, camera, scene_depth_texture, density_texture, history_texture, uniforms):
    """
    Renders a single pixel using volume ray marching with temporal accumulation.
    
    Args:
        uv: vec2, texture coordinates [0, 1]
        camera: object with position(), viewMatrix(), projectionMatrix()
        scene_depth_texture: sampler2D
        density_texture: sampler3D
        history_texture: sampler2D
        uniforms: dict containing stepSize, maximumSteps, densityScale, absorption, 
                  emissionStrength, fieldMode, temporalEnabled, historyWeight, 
                  frameIndex, jitterEnabled, selfShadowingEnabled, lightDirection, lightIntensity
    
    Returns:
        vec4, final color
    """
    # 1. Ray Setup
    clip_pos = vec4(uv * 2.0 - 1.0, 1.0, 1.0)
    inv_vp = inverse(camera.projectionMatrix() * camera.viewMatrix())
    far_world = vec3(inv_vp * clip_pos) / clip_pos.w
    origin = camera.position()
    direction = normalize(far_world - origin)
    
    # 2. Box Intersection [-1, 1]^3
    inv_direction = 1.0 / direction
    t1 = (-1.0 - origin) * inv_direction
    t2 = (1.0 - origin) * inv_direction
    t_min = max(min(t1.x, t1.y, t1.z), min(t2.x, t2.y, t2.z))
    t_max = min(max(t1.x, t1.y, t1.z), max(t2.x, t2.y, t2.z))
    
    # 3. Scene Clipping
    scene_depth = texture(scene_depth_texture, uv).r
    if scene_depth < 1.0:
        # Unproject scene point to get distance along ray
        clip_scene = vec4(uv * 2.0 - 1.0, scene_depth, 1.0)
        far_scene = vec3(inv_vp * clip_scene) / clip_scene.w
        scene_dist = dot(far_scene - origin, direction)
        if scene_dist > 0:
            t_max = min(t_max, scene_dist)
            
    if t_max <= t_min:
        return vec4(0.0) # No intersection
        
    # 4. Jittering
    t_start = max(t_min, 0.0)
    if uniforms['jitterEnabled']:
        jitter = compute_jitter(uv, uniforms['frameIndex'])
        t_start += jitter * uniforms['stepSize']
        
    # 5. Initialization
    accumulated = vec4(0.0)
    found_volume = False
    representative_position = vec3(0.0)
    
    # 6. Ray Marching Loop
    for i in range(uniforms['maximumSteps']):
        if t_start >= t_max:
            break
            
        pos = origin + direction * t_start
        density = texture(density_texture, pos * 0.5 + 0.5).r
        
        # Volume Entry Detection
        if not found_volume and density > 0.01:
            representative_position = pos
            found_volume = True
            
        # Color Computation
        smoke_color = vec3(0.0)
        emitted_light = vec3(0.0)
        transmittance = 1.0
        
        if uniforms['fieldMode'] == 0:
            # Smoke/Fire mode
            # Simplified: assume temperature/fire textures are sampled elsewhere or passed
            # Compute light transmittance if self-shadowing is enabled
            if uniforms['selfShadowingEnabled']:
                transmittance = compute_light_transmittance(pos, uniforms)
            ambient = vec3(0.12, 0.15, 0.20)
            direct = vec3(1.0, 0.88, 0.72) * uniforms['lightIntensity'] * transmittance
            # Mix smoke color based on position.y (simplified)
            smoke_color = mix(vec3(0.3, 0.04, 0.01), vec3(1.0, 0.65, 0.08), density)
            # Emitted light from fire (simplified)
            emitted_light = smoke_color * uniforms['emissionStrength']
        elif uniforms['fieldMode'] == 1:
            smoke_color = mix(vec3(0.3, 0.04, 0.01), vec3(1.0, 0.65, 0.08), density)
        elif uniforms['fieldMode'] in [2, 3]:
            density_abs = abs(density - 0.5) * 2.0
            smoke_color = vec3(1.0, 0.0, 0.0) if density >= 0.5 else vec3(0.0, 0.0, 1.0)
            
        # Accumulation
        alpha = 1.0 - exp(-density * uniforms['stepSize'])
        color_sample = smoke_color + emitted_light
        accumulated.rgb += color_sample * alpha * (1.0 - accumulated.a)
        accumulated.a += alpha * (1.0 - accumulated.a)
        
        t_start += uniforms['stepSize']
        
    # 8. Temporal Accumulation
    if uniforms['temporalEnabled']:
        history = texture(history_texture, uv).rgb
        out_color = mix(history, accumulated.rgb, uniforms['historyWeight'])
    else:
        out_color = accumulated.rgb
        
    return vec4(out_color, 1.0)

def compute_jitter(frag_coord, frame_index):
    """
    Computes a random jitter value based on fragment coordinates and frame index.
    """
    coord = frag_coord.xy
    value = fract(vec3(coord.xyx) * 0.1031)
    value += dot(value, value.yzx + 33.33 + float(frame_index) * 0.01)
    return fract((value.x + value.y) * value.z)

def compute_light_transmittance(pos, uniforms):
    """
    Computes light transmittance by marching towards the light source.
    """
    light_dir = normalize(vec3(uniforms['lightDirection']) + vec3(0.0001))
    optical_depth = 0.0
    step_size = 0.08
    for _ in range(20):
        # Simplified march direction
        march_pos = pos + light_dir * step_size * _ # Simplified loop
        # Check bounds
        if abs(march_pos.x) > 1.0 or abs(march_pos.y) > 1.0 or abs(march_pos.z) > 1.0:
            break
        density = texture(density_texture, march_pos * 0.5 + 0.5).r
        optical_depth += density * step_size
        
    return exp(-optical_depth * uniforms['absorption'] * uniforms['densityScale'] * 3.0)
```

### Parameters
- `gridSpacing`: float, default 0.5. Used for debug visualization of the grid [S1, unofficial].
- `densityScale`: float. Multiplier for density values in light transmittance calculation [S1, unofficial].
- `stepSize`: float. Distance advanced per ray march step [S1, unofficial].
- `maximumSteps`: int. Maximum number of ray march steps per ray (default 767) [S1, unofficial].
- `frameIndex`: int. Current frame number, used for jitter randomization [S1, unofficial].
- `jitterEnabled`: bool. Enables random offset to ray start distance [S1, unofficial].
- `adaptiveSteppingEnabled`: bool. Uniform present, but usage not explicitly shown in the core loop logic [S1, unofficial].
- `fieldMode`: int. Selects rendering mode: 0 (Smoke/Fire), 1 (Simple Smoke), 2/3 (Abstract/Color-coded) [S1, unofficial].
- `lightDirection`: vec3. Direction of the light source [S1, unofficial].
- `lightIntensity`: float. Intensity multiplier for the light [S1, unofficial].
- `absorption`: float. Absorption coefficient for Beer-Lambert law in transmittance [S1, unofficial].
- `emissionStrength`: float. Multiplier for emitted light from fire/heat [S1, unofficial].
- `selfShadowingEnabled`: bool. Enables shadow ray marching [S1, unofficial].
- `temporalEnabled`: bool. Enables blending with history texture [S1, unofficial].
- `historyWeight`: float. Weight for the previous frame's color when blending [S1, unofficial].

### Edge Cases and Pitfalls
- **Division by Zero**: The `intersectBox` function computes `1.0 / rayDirection`. If `rayDirection` is zero, this results in infinity/NaN. The code relies on the caller to provide valid normalized directions [S1, unofficial].
- **Light Direction Normalization**: An epsilon (0.0001) is added to the light direction before normalization to prevent issues if the light direction is exactly zero [S1, unofficial].
- **Density Texture Coordinate Mapping**: The density texture is sampled at `position * 0.5 + 0.5`, mapping the world space `[-1, 1]` box to texture space `[0, 1]`. Incorrect mapping leads to sampling outside the volume [S1, unofficial].
- **No Volume Intersection**: If the ray does not intersect the volume box or is occluded by scene geometry, the fragment is discarded. This must be handled to avoid rendering artifacts [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the maximum number of steps per ray. The self-shadowing adds an additional $O(S_{shadow})$ per sample if enabled.
- **Memory Complexity**: $O(R \cdot H)$ for the history texture, where $R$ is the resolution and $H$ is the channel size (4 bytes for RGBA). The density texture size depends on the simulation resolution.
- **Performance**: The method is designed for real-time performance, leveraging GPU parallelism. The use of temporal accumulation allows for fewer steps or lower resolution sampling while maintaining visual quality, reducing noise [S1, unofficial].
- **Strengths**: Reduces noise and flickering in dynamic volumes. Simple to implement with standard OpenGL/DirectX features. Supports various field modes for different visual styles.
- **Limitations**: Temporal accumulation can cause ghosting if the camera or volume moves rapidly without proper reprojection. The method assumes a canonical box for the volume, which may require transformation for arbitrary meshes. Self-shadowing is computationally expensive.
- **When to Use**: Suitable for real-time fluid simulations, smoke, and fire effects where noise reduction is critical and motion is relatively smooth.
- **When Not to Use**: Not suitable for static volumes where offline rendering is acceptable, or for highly dynamic scenes with rapid camera movement where ghosting artifacts are unacceptable without advanced reprojection techniques.

## Background
The method is part of the HZL Fluid Effects Studio, a tool for simulating and rendering fluid effects. It was developed by pvasbob and released as open-source software [S1, unofficial]. The technique builds upon standard volume ray marching principles, adding temporal accumulation to improve visual quality in real-time applications. It is used in the HZL Fluid Effects Studio for rendering smoke, fire, and abstract density fields.

## Sources
[S1] pvasbob/HZL-fluidEffectsStudio, https://github.com/pvasbob/HZL-fluidEffectsStudio, fetched 2026-10-27 (unofficial)
