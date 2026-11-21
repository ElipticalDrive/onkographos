# Cube Camera Volumetric Shadows


## Overview
A method to cast volumetric shadows in all directions from a single light source by using a CubeCamera to capture depth from six directions, overcoming the limitation of a single directional light camera [S1, unofficial]. Unlike the single-direction light camera approach which requires the light camera to point at the viewer to maximize shadow map coverage, the cube camera captures shadows in all directions but is considered a 'hack' with lower quality than other examples [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry, light position, light far plane distance (`shadowFar`), cube render target dimensions (`SHADOW_MAP_SIZE`).
*   **Output:** A cube texture containing normalized depth values in the red channel, used by the volumetric lighting effect's fragment shader.

**Data Structures:**
1.  **`WebGLCubeRenderTarget`**: Stores the depth data.
    *   Format: `RGBAFormat`
    *   Type: `FloatType`
    *   Mipmaps: Disabled (`generateMipmaps: false`)
    *   Filtering: Linear (`minFilter: LinearFilter`, `magFilter: LinearFilter`)
    *   Depth Buffer: Enabled (`depthBuffer: true`) [S1, unofficial]
2.  **`CubeCamera`**: Captures the scene from six orthogonal directions.
    *   Near plane: `CUBE_CAMERA_NEAR`
    *   Far plane: `CUBE_CAMERA_FAR`
    *   Target: The `WebGLCubeRenderTarget` [S1, unofficial]
3.  **`shadowMaterial`**: A custom `ShaderMaterial` used to render depth.
    *   Vertex Shader: Outputs `vWorldPosition` (world space position of the vertex).
    *   Fragment Shader: Computes normalized distance to the light.
    *   Uniforms: `lightPosition` (vec3), `shadowFar` (float) [S1, unofficial]

**Steps:**
1.  Create a `WebGLCubeRenderTarget` with the specified size, RGBA format, FloatType, no mipmaps, linear filtering, and depth buffer enabled [S1, unofficial].
2.  Initialize a `CubeCamera` with near and far clipping planes, targeting the created cube render target [S1, unofficial].
3.  Create a custom `shadowMaterial` that calculates the linear distance from the vertex's world position to the light position [S1, unofficial].
4.  Normalize the distance to the range $[0, 1]$ by dividing by the shadow far plane and clamping the result [S1, unofficial].
5.  Output the normalized distance to the red channel of the fragment color, leaving other channels zero [S1, unofficial].
6.  Replace all scene object materials with the `shadowMaterial` before rendering the cube camera snapshot [S1, unofficial].
7.  Render the scene using the `CubeCamera` to capture the depth texture [S1, unofficial].
8.  Restore the original scene materials after capturing the depth texture [S1, unofficial].
9.  Pass the resulting cube shadow map to the volumetric lighting effect's fragment shader to calculate shadows in all directions [S1, unofficial].

**Formulas:**
The normalized distance $d_{norm}$ for a fragment at world position $\mathbf{p}$ relative to light position $\mathbf{l}$ and far plane $F$ is:
$$ d_{norm} = \text{clamp}\left( \frac{\|\mathbf{p} - \mathbf{l}\|}{F}, 0.0, 1.0 \right) $$
In the provided implementation, the light position is assumed to be the origin or handled such that `length(vWorldPosition)` represents the distance [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Setup Render Target
shadow_cube_render_target = WebGLCubeRenderTarget(
    size=SHADOW_MAP_SIZE,
    format=RGBAFormat,
    type=FloatType,
    generate_mipmaps=False,
    min_filter=LinearFilter,
    mag_filter=LinearFilter,
    depth_buffer=True
)

# Step 2: Setup Cube Camera
shadow_cube_camera = CubeCamera(
    near=CUBE_CAMERA_NEAR,
    far=CUBE_CAMERA_FAR,
    target=shadow_cube_render_target
)

# Step 3: Define Shadow Material
shadow_material = ShaderMaterial(
    vertex_shader="""
        varying vec3 vWorldPosition;
        void main() {
            vec4 worldPosition = modelMatrix * vec4(position, 1.0);
            vWorldPosition = worldPosition.xyz;
            gl_Position = projectionMatrix * viewMatrix * worldPosition;
        }
    """,
    fragment_shader="""
        uniform vec3 lightPosition;
        uniform float shadowFar;
        varying vec3 vWorldPosition;
        void main() {
            // Calculate linear distance from the light source
            float distance = length(vWorldPosition - lightPosition);
            // Normalize distance to [0, 1] using the shadow camera's far plane
            float normalizedDistance = clamp(distance / shadowFar, 0.0, 1.0);
            // Store the normalized distance in the red channel.
            gl_FragColor = vec4(normalizedDistance, 0.0, 0.0, 1.0);
        }
    """,
    side=DoubleSide,
    uniforms={
        "lightPosition": {"value": light_position_vector},
        "shadowFar": {"value": CUBE_CAMERA_FAR}
    }
)

# Step 4: Render Depth
original_materials = []
for object in scene.objects:
    original_materials.append(object.material)
    object.material = shadow_material

# Render the scene from the cube camera's perspective
shadow_cube_camera.update(renderer, scene)

# Step 5: Restore Scene
for i, object in enumerate(scene.objects):
    object.material = original_materials[i]

# Step 6: Use Shadow Map
# Pass shadow_cube_render_target.texture to the volumetric lighting effect's fragment shader
```

**Edge Cases and Pitfalls:**
*   **No Native Cube Depth Texture:** Three.js does not provide a native `CubeDepthTexture`, requiring a custom depth extraction method using a color buffer [S1, unofficial].
*   **Flickering Artifacts:** Shadow beams may flicker due to shadow map resolution. This is a side-effect of the resolution and tends to happen in scenes with a large number of objects occluding the light, even with simple geometries. Increasing the resolution stabilizes the effect [S1, unofficial].
*   **Performance:** Scaling to multiple light sources requires extra code for each new source (new light camera, new shadow FBO, passing textures/matrices, calculating shadows/scattering/accumulation) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of objects in the scene for the material replacement and rendering pass. The final shadow calculation in the volumetric effect is linear with respect to the number of rays/march steps.
*   **Memory Complexity:** Proportional to $6 \times W \times H$ floats (where $W, H$ are the cube map face dimensions) for the shadow map, plus overhead for material swapping.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scaling to multiple light sources is described as "not much trouble besides requiring a lot of extra code for each new source of light" [S1, unofficial].
*   **Strengths:** Captures shadows in all directions from a single light source without needing to orient the light camera towards the viewer [S1, unofficial].
*   **Limitations:** Considered a 'hack' with lower quality than other examples [S1, unofficial]. Prone to flickering artifacts at lower resolutions [S1, unofficial].
*   **When to use:** When volumetric shadows from a point-like or omnidirectional light source are needed in a real-time post-processing pipeline, and other methods are not feasible.
*   **When not:** When high-quality shadows are required, or when performance is critical and the overhead of material swapping and cube rendering is prohibitive.

## Background
*   **Origin:** The method is described in a blog post by Maxime Heckel [S1, unofficial].
*   **History:** It is part of an exploration of volumetric lighting techniques, specifically addressing the limitation of single-direction light cameras [S1, unofficial].
*   **Usage:** Used in real-time volumetric lighting with post-processing and raymarching, particularly in WebGL/Three.js environments [S1, unofficial].

## Sources
[S1] On Shaping Light: Real-Time Volumetric Lighting with Post-Processing ..., https://blog.maximeheckel.com/posts/shaping-light-volumetric-lighting-with-post-processing-and-raymarching/, fetched 2026-10-28 (unofficial)
