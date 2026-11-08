# Billboard Voxelization

Also called: firstVoxelize, instanced billboard voxelization

## Overview
Billboard Voxelization is a two-pass rasterization-based technique that initializes a position map by rendering instanced quads (billboards) corresponding to voxel centers, writing the nearest world-space positions to a framebuffer [S1, unofficial]. It differs from [Position Map Voxelization](2604-position-map-voxelization.md) by explicitly rendering geometry to establish depth, and from [Hardware-accelerated surface voxelization via 2D textures](4042-hardware-accelerated-surface-voxelization-via-2d-textures.md) by using a full position map rather than just depth or surface normals [G].

## How it works
**Inputs:**
- A 3D volume grid of resolution $D \times D \times D$ [G].
- A bounding box defined by $p_{min}$ and $p_{max}$ [G].
- A light source with projection matrix $P$, view matrix $V$, and near plane distance [S1, unofficial].
- A pre-defined quad geometry (two triangles forming a square) [G].

**Outputs:**
- A Position Map Framebuffer Object (FBO) containing a color attachment (Position Map) and a depth attachment [S1, unofficial].
- The Position Map texture stores the world-space position of the nearest voxel surface point for each screen pixel [S1, unofficial].

**Data Structures:**
1. **Position Map FBO:**
   - **Color Attachment:** A 2D texture with internal format `GL_RGBA32F`, width and height matching window dimensions, `GL_NEAREST` filtering, and `GL_CLAMP_TO_EDGE` wrapping [S1, unofficial].
   - **Depth Attachment:** A renderbuffer with format `GL_DEPTH_COMPONENT`, width and height matching window dimensions [S1, unofficial].
2. **Instanced Quad VAO:**
   - Contains vertex positions (`vertPos`) and normals (`vertNor`) for a unit quad [S1, unofficial].
   - Contains instance attributes: `voxelPosition` (vec3, world-space center of the voxel), `voxelSize` (vec3, dimensions of the voxel), and `voxelData` (float, scalar value associated with the voxel) [S1, unofficial].

**Steps:**
1. **Initialize FBO:** Create the Position Map FBO with the specified color and depth attachments [S1, unofficial].
2. **Clear Buffers:** Bind the position FBO and clear color and depth buffers to 0 [S1, unofficial].
3. **Setup Matrices:** Load the light's projection matrix $P$, view matrix $V$, and a modified inverse view matrix $V_i$ where the translation components are zeroed out before transposing [S1, unofficial]. Load light near plane, clip distance, and volume position uniforms [S1, unofficial].
4. **Render Pass:** Bind the shader program and the instanced quad VAO. For each voxel in the volume:
   - Construct a model matrix $M$ using `voxelSize` and `voxelPosition`:
     $$ M = \begin{bmatrix} \text{voxelSize}_x & 0 & 0 & \text{voxelPosition}_x \\ 0 & \text{voxelSize}_y & 0 & \text{voxelPosition}_y \\ 0 & 0 & \text{voxelSize}_z & \text{voxelPosition}_z \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
   - Compute clip-space position:
     $$ \text{gl\_Position} = P \cdot V \cdot M \cdot \text{vec4}(\text{vertPos}, 1.0) $$
   - Output `voxelData` to the fragment shader [S1, unofficial].
   - In the fragment shader, output the world-space position of the fragment (reconstructed from depth and screen coordinates or passed from the vertex shader) [G].
   - The depth test ensures only the nearest voxel surface is written to the position map [S1, unofficial].
5. **Cleanup:** Unbind the VAO, volume, shader, and framebuffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def billboard_voxelize(volume, P, V, window_dims):
    # 1. Initialize FBO
    fbo = create_fbo(color_format='GL_RGBA32F', depth_format='GL_DEPTH_COMPONENT', size=window_dims)
    bind_fbo(fbo)
    clear_buffers(0.0)  # Color and depth buffers cleared to 0

    # 2. Setup matrices and uniforms
    V_inv = zero_translation(V)  # Zero translation components
    V_inv = transpose(invert(V_inv))  # Transpose before/invert as per source
    load_uniforms(P, V, V_inv, near_plane, clip_distance, volume_position)

    # 3. Render pass
    bind_shader(voxel_shader)
    bind_vao(instanced_quad_vao)
    for voxel in volume:
        M = identity_matrix()
        M[0][0] = voxel.voxelSize.x
        M[1][1] = voxel.voxelSize.y
        M[2][2] = voxel.voxelSize.z
        M[3][3] = 1.0
        M[3][0] = voxel.voxelPosition.x
        M[3][1] = voxel.voxelPosition.y
        M[3][2] = voxel.voxelPosition.z
        gl_Position = P @ V @ M @ vec4(voxel.vertPos, 1.0)
        output_voxel_data(voxel.voxelData)
    unbind_vao()
    unbind_volume()
    unbind_shader()
    unbind_fbo()
```

**Parameters:**
- $D$: Volume resolution [G].
- $W, H$: Window resolution [G].
- $P, V$: Light projection and view matrices [S1, unofficial].

**Edge Cases:**
- **Voxels outside frustum:** Voxels outside the light's view frustum are culled by the graphics pipeline [G].
- **Precision:** Using `GL_RGBA32F` for the position map ensures sufficient precision for world-space coordinates [S1, unofficial].
- **Degenerate Input:** If the volume is empty, the FBO remains cleared [G].

## Subtopics

### Position Map FBO Setup
Creates a Framebuffer Object (FBO) with a color attachment (position map) and a depth attachment for voxelization rendering. The color attachment is a 2D texture with internal format `GL_RGBA32F`, width and height matching window dimensions, nearest filtering, and clamp to edge wrapping. The depth attachment is a renderbuffer with format `GL_DEPTH_COMPONENT`, width and height matching window dimensions [S1, unofficial].

## Operating characteristics
- Time complexity: $O(N)$ where $N$ is the number of voxels [G].
- Memory complexity: $O(W \cdot H)$ for the position map and depth buffer, plus $O(N)$ for instance data [G].
- Performance: Not given in the sources.
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
