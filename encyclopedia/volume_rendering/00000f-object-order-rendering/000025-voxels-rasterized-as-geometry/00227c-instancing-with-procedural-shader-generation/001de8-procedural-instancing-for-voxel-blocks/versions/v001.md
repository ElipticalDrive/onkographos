# Procedural Instancing for Voxel Blocks

Also called: Cubic Grid Voxel Rendering with Instance Buffers, Feature/center instancing, GalensUnified Cubic Grid Renderer, Procedural Instancing, center instancing, voxel object generator

## Overview
Procedural instancing for voxel blocks replaces baked mesh geometry with a compact instance record per voxel, resolving position, color, and occlusion in the vertex shader using a shared cube template [S1, S2, S3, unofficial]. It differs from surface-based instancing by operating on the full cubic grid occupancy rather than extracted surfaces, and from 3D Gaussian splatting by using hard-edged geometric primitives rather than soft volumetric blobs [S1, S2, S3, unofficial].

## How it works
**Inputs:**
- Occupancy data: A list of occupied voxel coordinates.
- Texture atlas: A 2D texture array containing block face textures.
- Shared geometry: A static vertex buffer containing a 24-vertex cube template (6 faces × 4 vertices) [S1, S3, unofficial].

**Outputs:**
- Rasterized image of the voxel scene.

**Data Structures:**

1.  **Instance Record (Per Voxel):**
    The instance data is stored in a structured buffer (e.g., `StructuredBuffer` in HLSL or `std430` buffer in GLSL). The layout varies slightly by implementation but generally contains:
    -   **Position:** `vec3` or `float3` (x, y, z coordinates) [S2, S3, unofficial].
    -   **Texture Index:** `int` or `uint` identifying the texture in the atlas [S3, unofficial].
    -   **Shape/Rotation:** `uint` or packed `ushort` fields. In [S3, unofficial], this is a 32-bit `uint` where the lower 16 bits represent the `shape` index and the upper 16 bits represent the `rotation` index. The rotation is calculated as `(int)up * 4 + forward` [S3, unofficial].
    -   **Tint:** `vec3` or `float3` (r, g, b) [S3, unofficial].
    -   **Clip Parameter:** In some implementations (e.g., Unity), a `float4` position is used where the `w` component stores a clip distance [S2, unofficial].

    *GLSL `std430` Layout Example (32 bytes):*
    -   `posX`: float (4 bytes)
    -   `posY`: float (4 bytes)
    -   `posZ`: float (4 bytes)
    -   `textureIndex`: int (4 bytes)
    -   `shapeRotation`: uint (4 bytes)
    -   `tintR`: float (4 bytes)
    -   `tintG`: float (4 bytes)
    -   `tintB`: float (4 bytes)

    *Compact Layout Example (6 bytes per voxel):*
    -   Flat grid index (integer)
    -   Six-bit surviving-face mask (uint8)
    -   Orbit byte (uint8) [S1, unofficial].

2.  **Vertex Buffer (Shared):**
    -   Contains the vertices of a unit cube centered at the origin or (0,0,0) to (1,1,1).
    -   Total vertices: 24 (4 vertices per face × 6 faces) [S1, S3, unofficial].
    -   Each vertex includes position and UV coordinates [S3, unofficial].

**Algorithm Steps:**

1.  **Instance Buffer Population:**
    For each occupied voxel, generate an instance record containing its world position, texture ID, rotation/orientation, and tint color. Store these records in a GPU-accessible buffer [S2, S3, unofficial].

2.  **Draw Call:**
    Issue a single instanced draw call (e.g., `DrawInstanced` or `glDrawArraysInstanced`) using the shared 24-vertex cube template. The number of instances equals the number of occupied voxels [S1, S3, unofficial].

3.  **Vertex Shader Execution (Per Vertex per Instance):**
    The vertex shader transforms the shared cube vertices into world/screen space for each instance.

    *Step 3.1: Vertex Selection*
    Determine which vertex of the shared cube is being processed.
    $$ v_{index} = gl\_VertexID + (shapeRotation \ \& \ 0xFFFF) \times verticesPerShape $$
    Where `shapeRotation & 0xFFFF` extracts the lower 16 bits (shape index) [S3, unofficial].

    *Step 3.2: Rotation Matrix Retrieval*
    Retrieve the precomputed rotation matrix corresponding to the instance's orientation.
    $$ R = \text{RotationMatrixArray}[shapeRotation \gg 16] $$
    Where `shapeRotation >> 16` extracts the upper 16 bits (rotation index) [S3, unofficial].

    *Step 3.3: Position Transformation*
    Transform the local vertex position $v_{local}$ to world space:
    $$ v_{centered} = v_{local} - 0.5 $$
    $$ v_{rotated} = R \times v_{centered} $$
    $$ v_{translated} = v_{rotated} + 0.5 + P_{instance} + P_{chunk} $$
    Where $P_{instance}$ is the voxel position from the instance buffer, and $P_{chunk}$ is an optional chunk offset [S3, unofficial].

    *Step 3.4: Clip Space Conversion*
    Convert to homogeneous clip space:
    $$ v_{clip} = Projection \times View \times v_{translated} $$
    If a clip parameter $C$ is present in the instance data (e.g., Unity's `_Clip`), adjust the w-component:
    $$ v_{clip}.w = v_{clip}.w + C $$
    Or set a clip distance to discard hidden faces by moving corners outside the clip volume [S1, S2, unofficial].

    *Step 3.5: Output Attributes*
    Pass the texture index and tint color to the fragment shader [S3, unofficial].

4.  **Fragment Shader Execution:**
    Sample the texture atlas using the interpolated UV coordinates and the texture index from the vertex shader. Apply the tint color to the sampled albedo [S3, unofficial].

**Parameters:**
-   `verticesPerShape`: Uniform integer specifying the number of vertices per shape (typically 24 for a cube) [S3, unofficial].
-   `chunkPos`: Uniform `vec3` for chunk-based positioning [S3, unofficial].
-   `view`, `projection`: Uniform matrices for camera transformation [S3, unofficial].

**Edge Cases and Pitfalls:**
-   **Hidden Faces:** Instancing submits whole cubes. Hidden faces are culled by moving their vertices outside the clip volume in the vertex shader rather than omitting them from the draw call. This increases vertex processing load by 2.0× to 3.3× for zero-gap layouts but avoids the overhead of multiple draw calls or buffer rebuilds [S1, unofficial].
-   **Precision Drift:** GPU float32 arithmetic can cause drift. The drift is bounded to under $10^{-7}$ of the unit box, which is less than half a step of an 8-bit channel, ensuring visual correctness [S1, unofficial].
-   **Matrix Multiplication Order:** Care must be taken with row-major vs. column-major matrix storage. A workaround (e.g., `spvWorkaroundRowMajor`) may be needed to ensure correct transformation [S3, unofficial].
-   **CPU/GPU Precision Mismatch:** Axis center tables should be kept as `Float64Array` on the CPU to prevent precision drift, converting to `f32` only for GPU copy [S1, unofficial].

## Operating characteristics
-   **Memory Usage:** Significantly reduced compared to baked meshes. Memory per voxel is approximately 6 bytes (vs. over 1 kilobyte for six baked quads) [S1, unofficial]. At resolution R=64, specimen memory reduced from 42.0 MB to 649 KB [S1, unofficial].
-   **Performance:** Geometry build speed is 1.9–4.6× faster than pre-branch core methods. Whole build speed is 1.3–1.8× faster [S1, unofficial]. Maintains interactive frame rates (~60 FPS) at high voxel densities [S2, unofficial].
-   **Cache Efficiency:** Cache capacity increases from 6 specimens to ~394 specimens at R=64 before exhausting memory limits [S1, unofficial].
-   **Limitations:** Increased vertex work for dense, zero-gap layouts due to submitting whole cubes and culling in the shader [S1, unofficial].

## Background
This technique is used in projects such as the block-showroom by lardratboy and the GalensUnified Cubic Grid Renderer. It leverages GPU instancing capabilities to render large numbers of voxel blocks efficiently by minimizing data transfer and draw calls [S1, S3, unofficial].

## Sources
[S1] Feature/center instancing, https://github.com/lardratboy/block-showroom/pull/7, fetched 2026-10-26 (unofficial)
[S2] NotABottle/Voxel-Shader, https://github.com/NotABottle/Voxel-Shader, fetched 2026-10-26 (unofficial)
[S3] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
