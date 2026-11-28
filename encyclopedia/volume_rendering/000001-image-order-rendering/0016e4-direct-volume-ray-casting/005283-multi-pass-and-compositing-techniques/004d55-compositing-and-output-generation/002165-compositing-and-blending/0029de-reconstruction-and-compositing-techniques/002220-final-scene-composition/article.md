# Final Scene Composition


## Overview
Final Scene Composition is a post-processing technique that applies pre-computed volumetric data to an opaque scene [S1, unofficial]. It decouples volume calculation from surface rendering by sampling a 3D texture aligned with the scene geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Opaque scene depth buffer, fragment screen coordinates, linear depth, pre-computed 3D texture containing transmittance $T$ and in-scattered light $L_{in}$, and opaque scene color $C_{scene}$ [S1, unofficial].
*   **Output:** Final composited pixel color $C_{final}$ [S1, unofficial].

**Data Structures:**
*   **3D Texture:** A volumetric texture storing at least two components per voxel: transmittance $T$ and in-scattered light $L_{in}$ [S1, unofficial].

**Steps:**
1.  **Retrieve Depth:** For each fragment in the opaque scene pass, retrieve the linear depth $z_{linear}$ from the scene's depth buffer [S1, unofficial].
2.  **Convert Coordinates:** Convert the fragment's screen coordinates and linear depth into 3D texture coordinates $(u, v, w)$ using the exponential depth equation [S1, unofficial].
3.  **Sample Texture:** Sample the 3D texture at $(u, v, w)$ using hardware trilinear interpolation (`GL_LINEAR`) to obtain the transmittance $T$ and in-scattered light $L_{in}$ for that point in the volume [S1, unofficial].
4.  **Composite:** Combine the sampled volumetric data with the opaque scene color using the following equation [S1, unofficial]:
    $$
    C_{final} = C_{scene} \times T + L_{in}
    $$
    Where:
    *   $C_{scene}$ is the color of the opaque geometry at the fragment [S1, unofficial].
    *   $T$ is the transmittance sampled from the texture [S1, unofficial].
    *   $L_{in}$ is the in-scattered light sampled from the texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def final_scene_composition(scene_color, linear_depth, texture_3d):
    # 1. Retrieve linear depth from depth buffer (passed as input)
    z_linear = linear_depth
    
    # 2. Convert screen coordinates and linear depth to 3D texture coordinates (u, v, w)
    # The exact exponential depth equation is not provided in the sources [G].
    u, v, w = 0.0, 0.0, 0.0  # Coordinate conversion step omitted per sources
    
    # 3. Sample the integrated 3D texture using hardware trilinear interpolation
    T, L_in = texture_3d.sample_trilinear(u, v, w)
    
    # 4. Composite: Final Color = Scene Color * Transmittance + InScattered Light
    final_color = scene_color * T + L_in
    
    return final_color
```

**Parameters:**
*   **Texture Resolution:** Not given in the sources.
*   **Interpolation:** Trilinear (`GL_LINEAR`) is specified [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Opaque Scene Requirement:** This method requires a pre-existing opaque scene depth buffer for coordinate mapping [S1, unofficial].
*   **Coordinate Precision:** The accuracy of the composition depends on the precision of the depth-to-texture coordinate conversion [G].

## Subtopics

### Volumetric Lighting Composition
A variant that composites direct illumination, volumetric shadows, and caustic buffers [S2]. It renders surface caustics to a separate render target to allow for low-pass filtering before composition [S2].

### Scene Editing
A variant used in editable Gaussian splatting models (iVR-GS) [S3]. It allows real-time editing by replacing palette color $c_p$ with a new palette color $c'_p$, adjusting opacity by directly scaling the opacity term, and scaling attributes $k_a, k_d, k_s, \beta$ to adjust shading magnitudes [S3]. It renders with a new light direction using the Blinn-Phong equation based on primitive normals $n$ [S3].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for the composition step, as it involves a single texture lookup and arithmetic operation [G]. The overall complexity is dominated by the pre-computation of the 3D texture [G].
*   **Memory Complexity:** Depends on the resolution of the 3D texture, requiring storage for transmittance and in-scattered light volumes [G].
*   **Strengths:** High performance due to hardware interpolation and decoupling of volume and surface rendering [G].
*   **Limitations:** Limited to volumes that can be pre-computed into a 3D texture [G].

## Background
This method is described in the context of implementing volumetric fog in OpenGL using 3D textures [S1, unofficial]. It leverages hardware capabilities for 3D texture sampling to achieve real-time performance for participating media [S1, unofficial].

## Sources
[S1] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
[S2] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
[S3] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
