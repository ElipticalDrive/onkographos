# Final Scene Composition
Also called: Volumetric Fog Composition, 3D Texture Fog Blending

## Overview
Final Scene Composition is the post-processing or forward-lighting pass that integrates pre-computed volumetric data (such as in-scattered light and transmittance) into the opaque scene geometry. It solves the problem of rendering participating media (fog, smoke) over existing opaque surfaces without requiring per-pixel ray marching during the final image generation. It differs from iterative compositing methods by operating on a single lookup from a pre-integrated 3D texture rather than accumulating samples along a ray [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Opaque scene depth buffer (linear depth), screen coordinates of the fragment, pre-integrated 3D texture containing transmittance and in-scattered light, camera projection matrix, and view matrix.
*   **Outputs:** Final pixel color.

**Data Structures:**
*   **3D Texture:** A 3D texture volume storing the pre-integrated volumetric data. The dimensions typically correspond to the spatial extent of the volume or the view frustum. The texture format stores two components: Transmittance ($T$) and In-Scattered Light ($L_{in}$).
*   **Depth Buffer:** Linear depth values for opaque fragments.

**Steps:**
1.  **Retrieve Depth:** For each fragment being composited, retrieve the linear depth $d$ from the scene's depth buffer [S1, unofficial].
2.  **Compute 3D Coordinates:** Convert the fragment's screen coordinates $(x, y)$ and linear depth $d$ into 3D texture coordinates $(u, v, w)$ [S1, unofficial]. This conversion typically involves projecting the 3D position back into the volume's local space or using an exponential depth equation to map the view frustum to the texture space [S1, unofficial].
    *   *Note:* The specific exponential depth equation is not detailed in the source, but it maps the non-linear depth buffer to a linear 3D coordinate space suitable for trilinear interpolation [S1, unofficial].
3.  **Sample Texture:** Sample the 3D texture at $(u, v, w)$ using hardware trilinear interpolation (`GL_LINEAR`) to obtain the Transmittance $T$ and In-Scattered Light $L_{in}$ [S1, unofficial].
4.  **Composite:** Apply the sampled values to the original scene color $C_{scene}$ using the following formula [S1, unofficial]:
    $$
    C_{final} = C_{scene} \times T + L_{in}
    $$
    Where:
    *   $C_{final}$ is the final pixel color.
    *   $C_{scene}$ is the color of the opaque scene fragment.
    *   $T$ is the transmittance sampled from the texture (attenuation factor).
    *   $L_{in}$ is the in-scattered light sampled from the texture (additive lighting term).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def final_scene_composition(scene_color, linear_depth, screen_coords, 
                            volume_texture, view_matrix, proj_matrix):
    """
    Composites pre-integrated volumetric data onto the opaque scene.
    
    Args:
        scene_color: RGB color of the opaque fragment.
        linear_depth: Linear depth value from the depth buffer.
        screen_coords: (x, y) screen coordinates of the fragment.
        volume_texture: 3D texture containing (Transmittance, InScatteredLight).
        view_matrix: Camera view matrix.
        proj_matrix: Camera projection matrix.
        
    Returns:
        final_color: Computed final pixel color.
    """
    # 1. Convert screen coordinates and linear depth to 3D world position
    # Note: Exact projection inverse and depth conversion depend on the specific
    # exponential depth equation used. This is a conceptual representation.
    clip_coords = screen_to_clip(screen_coords, linear_depth, proj_matrix)
    world_pos = clip_to_world(clip_coords, view_matrix)
    
    # 2. Map world position to 3D texture coordinates (u, v, w)
    # This step assumes a specific mapping (e.g., exponential depth) defined
    # in the volume setup phase.
    u, v, w = world_to_volume_coords(world_pos)
    
    # 3. Sample the pre-integrated 3D texture
    # Hardware trilinear interpolation is used.
    transmittance, in_scattered_light = volume_texture.sample(u, v, w)
    
    # 4. Composite
    final_color = scene_color * transmittance + in_scattered_light
    
    return final_color
```

**Parameters:**
*   **Texture Filtering:** `GL_LINEAR` (trilinear) is specified for smooth interpolation [S1, unofficial].
*   **Depth Buffer:** Must contain linear depth, not normalized device coordinates [S1, unofficial].

**Edge Cases:**
*   **Out of Bounds:** If the 3D texture coordinates $(u, v, w)$ fall outside the volume bounds, the sampling behavior (clamp, repeat, or zero) must be defined. The source does not specify the default behavior.
*   **Precision:** The accuracy of the 3D coordinate conversion affects the alignment of the volumetric fog with the scene geometry.

## Subtopics

### Volumetric Lighting Composition
A variant that composites direct illumination, volumetric shadows, and caustic buffers. It renders surface caustics to a separate render target to allow for low-pass filtering before composition [S2]. This method is chosen when high-quality volumetric shadows and caustics are required, as opposed to simple fog [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment for the composition step, as it involves a single texture lookup and a few arithmetic operations [S1, unofficial].
*   **Memory Complexity:** Depends on the size of the pre-integrated 3D texture. The texture must store transmittance and in-scattered light for the entire volume [S1, unofficial].
*   **Strengths:** Very fast rendering as it avoids per-pixel ray marching. Leverages hardware texture filtering [S1, unofficial].
*   **Limitations:** Requires pre-computation of the 3D texture. Limited to static volumes or volumes that can be pre-integrated efficiently. Cannot handle dynamic lighting changes in real-time without re-computation [S1, unofficial].
*   **When to use:** Static or slowly changing volumetric effects like fog, smoke, or clouds where performance is critical [S1, unofficial].
*   **When not to use:** Dynamic volumetric effects with complex, time-varying lighting or geometry [S1, unofficial].

## Background
This method is described in the context of implementing volumetric fog using 3D textures in OpenGL [S1, unofficial]. It is also related to real-time volume caustics rendering, where volumetric lighting composition is used to combine direct illumination, shadows, and caustics [S2].

## Sources
[S1] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
[S2] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
