# Shadow Volume Voxelization

Also called: Shadow Volume Texture Generation

## Overview
Shadow Volume Voxelization is a technique to generate a 3D shadow volume texture by voxelizing scene geometry into a 3D grid, where each voxel stores a bitmask representing which of its 8 constituent sub-voxels are occluded [S1, unofficial]. It builds on [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md) by replacing the standard occupancy bit with a 3-bit occlusion mask per coarse voxel, enabling fine-grained shadow information for subsequent volume rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Scene geometry (meshes with model matrices), scene bounding box dimensions ($W, H, D$ in world units), and a scale factor (e.g., 10 units per meter) [S1, unofficial].
- Output: A 3D texture (`GL_R8UI`) with 3 mip levels (Level 0: $W \times H \times D$, Level 1: $W/2 \times H/2 \times D/2$, Level 2: $W/4 \times H/4 \times D/4$) [S1, unofficial].

**Data Structures:**
- `ShadowVolume` class containing:
  - `width`, `height`, `depth`: Integer dimensions of the finest mip level [S1, unofficial].
  - `volume_texture`: OpenGL 3D texture handle [S1, unofficial].
  - `shadow_volume_mip0`: Pointer to host-side memory for Level 0 (`uint8_t*`) [S1, unofficial].
  - `scene_xml`, `scene_root`: XML representation of the scene hierarchy [S1, unofficial].

**Steps:**

1. **Initialization:** Create a 3D texture with internal format `GL_R8UI` (unsigned byte, red channel only) and 3 mip levels [S1, unofficial]. Initialize Level 0 memory (`shadow_volume_mip0`) to zero [S1, unofficial].

2. **Voxel Processing:** For each voxel $(x_v, y_v, z_v)$ in the input shape:
   - Calculate world position:
     $$ \text{voxel\_pos} = \text{vec3}(x_v, y_v, z_v) + \text{vec3}(0.5, 0.5, 0.5) $$
     $$ \text{voxel\_pos} = \text{vec3}(\text{model\_matrix} \times \text{vec4}(\text{voxel\_pos}, 10.0)) $$
     (Assuming 10 units per meter scale) [S1, unofficial].
   - Map to grid coordinates $(x, y, z)$:
     $$ x = \lfloor \text{voxel\_pos}.x \rfloor + \text{width} / 2 $$
     $$ y = \lfloor \text{voxel\_pos}.y \rfloor $$
     $$ z = \lfloor \text{voxel\_pos}.z \rfloor + \text{depth} / 2 $$
   - **Boundary Check:** If $x < 0$ or $x \ge \text{width}$ or $y < 0$ or $y \ge \text{height}$ or $z < 0$ or $z \ge \text{depth}$, skip the voxel [S1, unofficial].

3. **Bitmask Calculation:**
   - Calculate Level 0 index for the coarse 2x2x2 block:
     $$ \text{index} = (x / 2) + \text{width} \times ((y / 2) + \text{height} \times (z / 2)) $$
     (Integer division) [S1, unofficial].
   - Calculate sub-voxel bit offset within the 2x2x2 block:
     $$ \text{offset} = (x \% 2) + 2 \times (y \% 2) + 4 \times (z \% 2) $$
   - Update the bitmask:
     $$ \text{shadow\_volume\_mip0}[\text{index}] \ |= \ (1 \ll \text{offset}) $$
     (Bitwise OR to accumulate occlusion bits) [S1, unofficial].

4. **Mipmap Generation:**
   - **Level 1:** Iterate over Level 0 dimensions $(i, j, k)$. Compute Level 1 index:
     $$ \text{index\_mip1} = (i / 2) + \text{width\_mip1} \times ((j / 2) + \text{height\_mip1} \times (k / 2)) $$
     $$ \text{shadow\_volume\_mip1}[\text{index\_mip1}] \ |= \ \text{shadow\_volume\_mip0}[\text{index}] $$
   - **Level 2:** Iterate over Level 1 dimensions $(i, j, k)$. Compute Level 2 index:
     $$ \text{index\_mip2} = (i / 2) + \text{width\_mip2} \times ((j / 2) + \text{height\_mip2} \times (k / 2)) $$
     $$ \text{shadow\_volume\_mip2}[\text{index\_mip2}] \ |= \ \text{shadow\_volume\_mip1}[\text{index}] $$

**Parameters:**
- Texture filtering: Min filter `GL_NEAREST_MIPMAP_NEAREST`, Mag filter `GL_NEAREST` [S1, unofficial].
- Wrap modes: `GL_CLAMP_TO_BORDER` for S, T, R [S1, unofficial].
- Mip range: Base level 0, Max level 2 [S1, unofficial].

**Pitfalls:**
- The shadow volume fragment shader is empty, indicating it may be used for depth testing or stencil operations rather than direct rendering [S1, unofficial].
- Precision issues may arise from floating-point to integer conversion in grid coordinate mapping [G].

## Subtopics

### Directional Shadow Map Volume Generation
A variant that generates shadow volumes specifically for directional lights by rendering scene geometry from a light-aligned orthographic camera into a 3D texture [S2, unofficial]. It checks a `ContinousVoxelizationPass` setting; if false, it skips rendering [S2, unofficial]. It calculates a tight orthographic frustum by transforming the scene bounding box corners into light space, applying a `LIGHT_CAMERA_OFFSET` of 50.0f to near/far planes [S2, unofficial]. It enables depth testing, blending `GL_ONE, GL_ZERO`, and front-face culling [S2, unofficial]. It iterates scene objects, skipping those not in `DEFERRED_STAGE` or without colliders, and performs frustum vs primitive culling before drawing to the shadow map volume [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Originated in the context of voxel-based real-time rendering engines, specifically as part of the Voxel-Render project [S1, unofficial]. It is used to provide shadow information for volume rendering by voxelizing shadow volumes into a 3D texture [S1, unofficial].

## Sources
[S1] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
[S2] maritim/Voxel-Cone-Tracing, https://github.com/maritim/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
