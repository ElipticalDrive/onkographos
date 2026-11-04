# Two-Pass Depth Ray Casting

Also called: Depth Peeling Ray Casting, Entry/Exit Depth Ray Casting, GPU Ray-Casting, Krüger Ray-Casting, Ray-Casting de Krüger, Volumetric Ray-Casting

## Overview

Two-Pass Depth Ray Casting is a direct volume rendering technique that computes ray entry and exit points by rendering the volume's bounding box twice with different face culling and depth test modes, then uses these coordinates to cast rays through the volume in a subsequent pass [S2, unofficial]. It solves the problem of efficiently determining the start and end of the integration interval for each pixel on the GPU without complex ray-volume intersection algorithms [S3, unofficial]. Unlike [Van Hook-Levoy Ray-Casting](3594-van-hook-levoy-ray-casting.md) which may use different traversal strategies, this method relies on hardware rasterization of the bounding box to find intersections [S1, unofficial]. It differs from [Koyamada's Algorithm](4148-koyamada-s-algorithm.md) by explicitly separating the depth determination phase from the accumulation phase using multiple render targets [S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- 3D volume texture (`t_Volume`): Scalar field data [S2, unofficial].
- Transfer function texture (`t_TFFunc`): Maps volume values to RGBA [S2, unofficial].
- Volume resolution (`t_DIM`): Dimensions in voxels (e.g., $m\_DataRes$) [S2, unofficial].
- Value span (`v\_Min`, `v\_Max`): Minimum and maximum values in the dataset [S2, unofficial].
- Bounding box geometry: Cube defined by 8 vertices and 36 indices [S2, unofficial].

**Outputs:**
- Final rendered image in `RenderFB` [S2, unofficial].

### Data Structures

**Bounding Box Geometry:**
- Vertices: 8 vertices, each with 3 floats for position $(x,y,z)$ and 3 floats for color $(r,g,b)$ [S2, unofficial].
- Indices: 36 `uint32`s forming 12 triangles (2 per face) [S2, unofficial].

**Framebuffers:**
- Three framebuffers (`m_ProjInFB`, `m_ProjOutFB`, `m_RenderFB`) resized to viewport dimensions $(m\_ViewportSize.x, m\_ViewportSize.y)$ [S2, unofficial].

**Render Targets (Textures):**
- `ProjInFB` (Entry): Floating-point texture, width/height matching window, `NearestFilter`, `ClampToEdgeWrapping`, `FloatType`, no mipmaps [S3, unofficial]. Stores world-space coordinates of the back-face intersection (ray entry) [S1, unofficial].
- `ProjOutFB` (Exit): Floating-point texture, same properties as `ProjInFB` [S3, unofficial]. Stores world-space coordinates of the front-face intersection (ray exit) [S2, unofficial].
- `3D Texture` (`cubeTex`): `Data3DTexture` with dimensions $(imageStackLength, 256, 256)$, `RedFormat`, `LinearFilter`, no mipmaps [S3, unofficial].

### Steps

1.  **Pass 1 (Entry Point):** Render the volume's bounding box with **back-face culling** enabled. Set the depth test to `DEPTH_LESS` and clear the depth buffer to 1.0. The fragment shader outputs the world-space coordinates of the intersection to `ProjInFB` [S2, unofficial]. This captures the nearest intersection (ray entry point) for each screen pixel [S2, unofficial].
2.  **Pass 2 (Exit Point):** Render the same bounding box with **front-face culling** enabled. Set the depth test to `DEPTH_GREATER` and clear the depth buffer to 0.0. The fragment shader outputs the world-space coordinates to `ProjOutFB` [S2, unofficial]. This captures the farthest intersection (ray exit point) [S2, unofficial].
3.  **Pass 3+ (Ray Casting and Accumulation):** Render a fullscreen quad (or front-facing cube) to `RenderFB`. Bind `ProjInFB` and `ProjOutFB` as textures providing ray start and end coordinates. Bind the 3D volume texture and transfer function.
    -   Calculate ray vector: $\vec{ray} = \vec{backPosition} - \vec{frontPosition}$ [S3, unofficial].
    -   Calculate ray length: $L = \|\vec{ray}\|$ [S3, unofficial].
    -   Calculate step parameters: $\delta = 1.0 / \text{steps}$, $\vec{deltaDirection} = \text{normalize}(\vec{ray}) \cdot \delta$, $\Delta L = \|\vec{deltaDirection}\|$ [S3, unofficial].
    -   Initialize: $\vec{currentPosition} = \vec{frontPosition}$, $\vec{accumulatedColor} = (0,0,0,0)$, $L_{acc} = 0$ [S3, unofficial].
    -   Loop for `steps` iterations (or up to a fixed max, e.g., 998):
        -   Sample volume: $I = \text{texture}(t\_Volume, \vec{currentPosition}).r$ [S3, unofficial].
        -   Apply contrast/brightness: $I' = \text{contrast} \cdot I + \text{brightness}$ [S3, unofficial].
        -   Threshold: If $I' \le \text{colorTreshLow}$ or $I' \ge \text{colorTreshHigh}$, set $I' = 0$ [S3, unofficial].
        -   Compute color contribution:
            -   If $I' \ge 0.5$: $C = (I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \delta, 0, 0, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \delta)$ [S3, unofficial].
            -   Else: $C = (I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \delta, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \delta, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \delta, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \delta)$ [S3, unofficial].
        -   Accumulate: $\vec{accumulatedColor} += C$ [S3, unofficial].
        -   Update: $L_{acc} += \Delta L$, $\vec{currentPosition} += \vec{deltaDirection}$ [S3, unofficial].
        -   Early Termination: If $L_{acc} \ge L$ or $\vec{accumulatedColor}.a \ge 1.0$, break loop [S3, unofficial].
    -   Output $\vec{accumulatedColor}$ to `gl_FragColor` [S3, unofficial].
4.  **Early Depth Test Optimization:** In passes 3 to N, write a depth value that forces fragment rejection in subsequent passes for fragments that have reached maximum opacity, utilizing the GPU's Early Depth Test feature to skip expensive fragment shader execution [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Entry Point
def pass1_entry_point():
    set_depth_test(LESS)
    clear_depth(1.0)
    set_culling(BACK)
    render_bounding_box()
    # Fragment shader: gl_FragColor = vec4(worldSpaceCoords, 1.0)

# Pass 2: Exit Point
def pass2_exit_point():
    set_depth_test(GREATER)
    clear_depth(0.0)
    set_culling(FRONT)
    render_bounding_box()
    # Fragment shader: gl_FragColor = vec4(worldSpaceCoords, 1.0)

# Pass 3+: Ray Casting
def pass3_ray_casting():
    set_depth_test(LESS) # For early rejection
    clear_depth(0.0)
    bind_framebuffer(RenderFB)
    bind_texture("tex", ProjInFB) # Entry coords
    bind_texture("texOut", ProjOutFB) # Exit coords (if needed, though entry is usually sufficient for direction if origin is known, but here we use both)
    bind_texture("cubeTex", t_Volume)
    
    # Render fullscreen quad
    render_quad()

    # Fragment Shader Logic
    def fragment_shader():
        # Get projected coordinates
        projected_coords = get_fragment_position()
        texc = vec2(
            (projected_coords.x / projected_coords.w + 1.0) / 2.0,
            (projected_coords.y / projected_coords.w + 1.0) / 2.0
        )
        
        # Retrieve entry and exit points
        back_position = texture2D(tex, texc).xyz
        front_position = get_world_space_coords() # From vertex shader/quad geometry
        
        # Edge case: Invalid intersection
        if back_position.x == 0.0 and back_position.y == 0.0:
            return vec4(0.0)
        
        # Ray setup
        ray = back_position - front_position
        ray_length = length(ray)
        steps = 256 # Typical value
        delta = 1.0 / steps
        delta_direction = normalize(ray) * delta
        delta_length = length(delta_direction)
        
        current_position = front_position
        accumulated_color = vec4(0.0)
        accumulated_length = 0.0
        
        # Parameters
        contrast = 1.0
        brightness = 0.0
        color_tresh_low = 0.0
        color_tresh_high = 1.0
        alpha_correction = 2.0
        
        for i in range(998): # Fixed max iterations
            # Sample volume
            texture_intensity = texture(cubeTex, current_position).r
            texture_intensity = contrast * texture_intensity + brightness
            
            # Thresholding
            if texture_intensity <= color_tresh_low or texture_intensity >= color_tresh_high:
                texture_intensity = 0.0
            
            # Color contribution
            if texture_intensity >= 0.5:
                color = vec4(
                    texture_intensity * 2.56 * alpha_correction * delta,
                    0.0,
                    0.0,
                    texture_intensity * 2.56 * alpha_correction * delta
                )
            else:
                color = vec4(
                    texture_intensity * 2.56 * alpha_correction * delta,
                    texture_intensity * 2.56 * alpha_correction * delta,
                    texture_intensity * 2.56 * alpha_correction * delta,
                    texture_intensity * 2.56 * alpha_correction * delta
                )
            
            accumulated_color += color
            accumulated_length += delta_length
            current_position += delta_direction
            
            # Early termination
            if accumulated_length >= ray_length or accumulated_color.a >= 1.0:
                break
        
        # Write depth for early rejection in subsequent passes if fully opaque
        if accumulated_color.a >= 1.0:
            set_depth(1.0) # Force rejection
        
        return accumulated_color
```

### Parameters
- `steps`: Number of steps along the ray. Default: 256 [S3, unofficial].
- `t_DIM` (`vec3`): Volume resolution in voxels [S2, unofficial].
- `v_Min`, `v_Max` (`float`): Min/Max volume values [S2, unofficial].
- `t_CoordIn`, `t_CoordOut` (`sampler2D`): Textures for entry/exit coordinates [S2, unofficial].
- `t_Volume` (`sampler3D`): 3D volume texture [S2, unofficial].
- `t_TFFunc` (`sampler1D`/`sampler2D`): Transfer function [S2, unofficial].
- `colorTreshLow`, `colorTreshHigh`: Intensity threshold bounds [S3, unofficial].
- `alphaCorrection`: Scalar multiplier for intensity [S3, unofficial].
- `contrast`, `brightness`: Contrast/brightness adjustment [S3, unofficial].

### Pitfalls
- **Scaling:** Performance is greatly impacted by data volume due to the rasterization of support primitives [S1, unofficial].
- **Zero Viewport:** Invalid framebuffer operations if viewport size is zero. Check `m_ViewportSize.x > 0.0 && m_ViewportSize.y > 0.0` before resizing [S2, unofficial].
- **Invalid Intersection:** If `backPosition` is `(0,0,0)`, it indicates an invalid intersection (edge of cube). Return black [S3, unofficial].
- **Accumulation Method:** This implementation uses simple additive accumulation without explicit opacity weighting $(1.0 - \text{accumulatedAlpha})$, unlike standard front-to-back alpha compositing [S3, unofficial].

## Operating characteristics

- **Time Complexity:** $O(P \cdot S)$, where $P$ is the number of pixels and $S$ is the number of steps per ray. The method requires 3+ passes per frame (2 for depth, 1+ for accumulation).
- **Memory Complexity:** $O(V + P)$, where $V$ is the volume texture size and $P$ is the framebuffer size for storing entry/exit coordinates [S2, unofficial].
- **Performance:** Early Depth Test allows skipping fragments that reach full opacity, improving performance in transparent regions [S1, unofficial]. However, the method does not scale well with large data volumes due to the overhead of rasterizing bounding box primitives for every pass [S1, unofficial].
- **Strengths:** Simple implementation on GPU; leverages hardware depth testing; no complex ray-triangle intersection code required [S1, unofficial].
- **Limitations:** Fixed step count may lead to aliasing or slow convergence; additive accumulation is less physically accurate than alpha compositing; poor scalability with data size [S1, S3, unofficial].
- **When to use:** Real-time rendering on Shader Model 2.0/3.0 GPUs for moderate-sized regular voxel grids [S1, unofficial].
- **When not to use:** Large-scale datasets requiring adaptive sampling or complex scene composition with surfaces [S1, unofficial].

## Background

- **Origin:** First proposed by Krüger [KW03] for Shader Model 2.0 GPUs [S1, unofficial].
- **Development:** Extended by Scharsach [Sch05] using Shader Model 3.0 features, including dynamic branching and block-based volume partitioning for selective memory loading [S1, unofficial].
- **Usage:** Used in interactive volume exploration tools, particularly for medical imaging (DICOM data) where regular voxel grids are common [S3, unofficial].

## Sources

[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] hands133/opengl-directVolumeRenderer, https://github.com/hands133/opengl-directVolumeRenderer, fetched 2026-10-26 (unofficial)
[S3] arisetiawan4601/dicom-volume-renderer, https://github.com/arisetiawan4601/dicom-volume-renderer, fetched 2026-10-27 (unofficial)
