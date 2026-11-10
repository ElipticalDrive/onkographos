# Billboard Voxelization

Also called: firstVoxelize, instanced billboard voxelization

## Overview
Billboard Voxelization is a two-pass rasterization-based technique that initializes a position map by rendering instanced quads (billboards) corresponding to voxel centers, writing the nearest world-space positions to a framebuffer. It differs from [Position Map Voxelization](2604-position-map-voxelization.md) by explicitly rendering geometry to establish depth, and from [Hardware-accelerated surface voxelization via 2D textures](4042-hardware-accelerated-surface-voxelization-via-2d-textures.md) by using a full position map rather than just depth or surface normals.

## How it works
**Inputs:**
- A 3D volume grid of resolution $D \times D \times D$.
- A bounding box defined by $p_{min}$ and $p_{max}$.
- A light source with projection matrix $P$, view matrix $V$, and near plane distance.
- A pre-defined quad geometry (two triangles forming a square).

**Outputs:**
- A Position Map Framebuffer Object (FBO) containing a color attachment (Position Map) and a depth attachment.
- The Position Map texture stores the world-space position of the nearest voxel surface point for each screen pixel.

**Data Structures:**
1.  **Position Map FBO:**
    -   **Color Attachment:** A 2D texture with internal format `GL_RGBA32F`. Dimensions match the window resolution (width $W$, height $H$). Filtering is `GL_NEAREST`, wrapping is `GL_CLAMP_TO_EDGE`.
    -   **Depth Attachment:** A renderbuffer with format `GL_DEPTH_COMPONENT`. Dimensions match the window resolution.
2.  **Instanced Quad VAO:**
    -   Contains vertex positions for a unit quad.
    -   Contains instance attributes: `voxelPosition` (vec3, world-space center of the voxel), `voxelSize` (vec3, dimensions of the voxel), and `voxelData` (float, scalar value associated with the voxel).

**Steps:**
1.  **Initialize FBO:** Create the Position Map FBO with the specified color and depth attachments. Bind the FBO.
2.  **Clear Buffers:** Clear the color buffer to $(0, 0, 0, 0)$ and the depth buffer to $1.0$ (or $0.0$ depending on depth function convention, typically $1.0$ for standard depth testing where closer is smaller). [S1, unofficial]
3.  **Setup Matrices:**
    -   Load the light's projection matrix $P$.
    -   Load the light's view matrix $V$.
    -   Compute a modified inverse view matrix $V_i$. This is done by zeroing out the translation components of $V$, inverting the resulting rotation/scaling matrix, and transposing it (or simply extracting the rotation part and transposing it, as the inverse of a rotation matrix is its transpose). [S1, unofficial]
    -   Load light near plane distance, clip distance, and volume position uniforms.
4.  **Render Pass:**
    -   Bind the Position Map FBO.
    -   Bind the shader program containing the vertex shader logic.
    -   Bind the instanced quad VAO.
    -   For each voxel in the volume:
        -   Calculate the world-space position of the voxel center $c$ using the bounding box and voxel index $i$:
            $$ c = p_{min} + \frac{i + 0.5}{D} (p_{max} - p_{min}) $$
        -   Construct a model matrix $M$ for the billboard:
            $$ M = \begin{bmatrix} s_x & 0 & 0 & c_x \\ 0 & s_y & 0 & c_y \\ 0 & 0 & s_z & c_z \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
            where $s_x, s_y, s_z$ are the voxel dimensions (often uniform, $s_x=s_y=s_z = \frac{p_{max}-p_{min}}{D}$). [S1, unofficial]
        -   Compute the clip-space position:
            $$ gl\_Position = P \cdot V \cdot M \cdot \begin{bmatrix} v_x \\ v_y \\ v_z \\ 1 \end{bmatrix} $$
            where $(v_x, v_y, v_z)$ are the local coordinates of the quad vertex. [S1, unofficial]
        -   Output the voxel data (e.g., density or material ID) to the fragment shader.
    -   In the fragment shader, output the world-space position of the fragment. This is typically computed as:
        $$ pos_{world} = V_i \cdot gl\_Position.xyw $$
        (Note: The exact reconstruction depends on the depth buffer and projection. A common method for position maps is to use the depth value $z_{ndc}$ and screen coordinates to reconstruct the view-space position, then transform to world-space. However, the source code suggests passing `voxelData` and using the vertex shader to compute `gl_Position`. The fragment shader must output the world position. A robust way is to pass the world position of the fragment from the vertex shader or reconstruct it from depth. The source code snippet only shows `gl_Position` calculation. We assume the fragment shader outputs the world position of the rendered billboard surface.) [G]
    -   The depth test ensures only the nearest voxel surface is written to the position map.
5.  **Cleanup:** Unbind the VAO, volume textures, shader, and framebuffer.

**Parameters:**
-   $D$: Volume resolution (e.g., 128, 256, 512).
-   $W, H$: Window resolution.
-   $P, V$: Light projection and view matrices.

**Edge Cases:**
-   **Voxels outside frustum:** Voxels outside the light's view frustum are culled by the graphics pipeline.
-   **Precision:** Using `GL_RGBA32F` for the position map ensures sufficient precision for world-space coordinates. [S1, unofficial]
-   **Degenerate Input:** If the volume is empty, the FBO remains cleared.

## Subtopics

### Position Map FBO Setup
Creates a Framebuffer Object (FBO) with a color attachment (position map) and a depth attachment for voxelization rendering. The color attachment is a 2D texture with internal format `GL_RGBA32F`, width and height matching window dimensions, nearest filtering, and clamp to edge wrapping. The depth attachment is a renderbuffer with format `GL_DEPTH_COMPONENT`, width and height matching window dimensions. [S1, unofficial]

## Operating characteristics
-   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is rendered as a billboard.
-   **Memory Complexity:** $O(W \cdot H)$ for the position map and depth buffer, plus $O(N)$ for the instance data.
-   **Performance:** Performance scales with the number of voxels and the window resolution. High-resolution volumes and large windows increase memory bandwidth usage.
-   **Strengths:** Simple to implement using standard graphics APIs; handles arbitrary voxel shapes via billboard geometry; provides a full position map for subsequent ray marching.
-   **Limitations:** Requires rendering all voxels, which can be slow for very high-resolution volumes; depth fighting may occur if voxels are coplanar or very close; requires careful handling of the position reconstruction in the fragment shader.
-   **When to use:** When a full position map is needed for advanced volume rendering techniques (e.g., ray marching with position-based shading) and the volume resolution is moderate.
-   **When not to use:** For very high-resolution volumes where performance is critical, or when only a depth map is sufficient.

## Background
Billboard Voxelization is used in real-time cloud rendering and other volume visualization applications. It was implemented in the "Cloud-Renderer" project by Jaafersheriff, which demonstrates a real-time volumetric cloud rendering pipeline. [S1, unofficial]

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
