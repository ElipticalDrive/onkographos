# Billboard Voxelization

Also called: firstVoxelize, instanced billboard voxelization

## Overview
Billboard Voxelization is a two-pass rasterization-based technique that converts a triangle mesh into a voxel grid by rendering instanced quads (billboards) at each voxel center. It solves the problem of mapping surface geometry to a discrete volume representation by writing the nearest voxel positions to a position map framebuffer object (FBO) [S1, unofficial]. It differs from siblings like [3D Grid Rasterization](903-3d-grid-rasterization.md) by using instanced geometry per voxel rather than rasterizing triangles directly into a 3D grid, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on standard 2D rasterization of billboards rather than compute shaders [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Triangle mesh vertices, voxel grid dimensions ($dim$), voxel size ($voxelSize$), voxel world positions ($voxelPosition$), camera/light projection ($P$) and view ($V$) matrices.
- Output: A 3D volume texture (or position map) where occupied voxels store their world position and data attributes.

**Data Structures:**
1. **Instanced Quad Buffer:** A VAO containing a single quad (4 vertices, 3 floats each for position).
2. **Instance Data Buffer:** A buffer of size $dim^3$ containing `vec3` voxel positions and `float` voxel data for each voxel in the grid [S1, unofficial].
3. **Position Map FBO:** A framebuffer with:
   - Color attachment: 2D texture, format `GL_RGBA32F`, dimensions matching the window/camera viewport, nearest filtering, clamp-to-edge wrapping [S1, unofficial].
   - Depth attachment: Renderbuffer, format `GL_DEPTH_COMPONENT`, matching viewport dimensions [S1, unofficial].

**Steps:**
1. **Initialize FBO:** Bind the position FBO. Clear color and depth buffers to 0 [S1, unofficial].
2. **Setup Matrices:** Load the projection matrix $P$ and view matrix $V$. Compute a modified inverse view matrix $V_i$ by zeroing the translation components of $V$ before transposing [S1, unofficial]. Load uniforms: $P$, $V$, $voxelSize$, and $voxelPosition$ [S1, unofficial].
3. **Bind Geometry:** Bind the instanced quad VAO and the instance data buffer. Enable vertex attribute array for instance positions (location 2) [S1, unofficial].
4. **Render Pass:** Draw the instanced quads using `glDrawArraysInstanced`. The vertex shader transforms the quad into world space using the instance position and scales it by $voxelSize$. The fragment shader writes the voxel data and position to the position map FBO [S1, unofficial].
5. **Cleanup:** Unbind the VAO, volume, shader, and framebuffer [S1, unofficial].

**Vertex Shader Logic:**
The vertex shader constructs a model matrix $M$ from the instance data:
$$
M = \begin{bmatrix}
voxelSize.x & 0 & 0 & voxelPosition.x \\
0 & voxelSize.y & 0 & voxelPosition.y \\
0 & 0 & voxelSize.z & voxelPosition.z \\
0 & 0 & 0 & 1
\end{bmatrix}
$$
The final position is computed as:
$$
gl\_Position = P \times V \times M \times vec4(vertPos, 1.0)
$$
Where $vertPos$ is the local quad vertex position [-1, 1] [S1, unofficial].

**Parameters:**
- $dim$: Voxel grid dimension (e.g., 64, 128, 256) [S1, unofficial].
- $voxelSize$: Physical size of a voxel in world units [S1, unofficial].
- $P, V$: Camera matrices [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Using `GL_RGBA32F` for the position map ensures high precision for world coordinates [S1, unofficial].
- **Depth Testing:** Standard depth testing is used to ensure only the nearest voxel position is written if billboards overlap (though in a regular grid, they typically do not overlap in screen space unless the camera is inside the volume) [S1, unofficial].
- **Matrix Inversion:** The modified inverse view matrix $V_i$ is used in some variants to transform normals or positions correctly without translation interference [S1, unofficial].

## Subtopics

### Position Map FBO Setup
Creates a Framebuffer Object (FBO) with a color attachment (position map) and a depth attachment for voxelization rendering. The color attachment is a 2D texture with internal format `GL_RGBA32F`, width and height matching window dimensions, nearest filtering, and clamp to edge wrapping. The depth attachment is a renderbuffer with format `GL_DEPTH_COMPONENT`, matching window dimensions [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ where $N = dim^3$ is the number of voxels, as each voxel is processed once via instancing.
- Memory: $O(dim^3)$ for the instance data buffer and the position map texture [S1, unofficial].

**Measured Performance:**
Performance depends on the GPU's ability to handle instanced rendering and the resolution of the position map. High-resolution grids ($dim > 256$) may require significant VRAM for the position map [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Simple to implement using standard rasterization; efficient for regular grids; leverages GPU instancing.
- Limitations: Requires a regular voxel grid; may produce artifacts if the camera is too close to the volume or if depth precision is insufficient; not suitable for hierarchical or sparse voxel structures without modification [S1, unofficial].

**When to Use:**
Use when a regular voxel grid is acceptable and hardware rasterization is preferred over compute shaders. Not suitable for sparse or hierarchical volumes [S1, unofficial].

## Background
Originated in the context of cloud rendering and real-time volume visualization, specifically in the "Cloud-Renderer" project by Jaafersheriff [S1, unofficial]. It is used for generating voxel data for volumetric clouds and other atmospheric effects where a regular grid structure is beneficial [S1, unofficial].

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
