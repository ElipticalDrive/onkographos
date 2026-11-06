# GPU Instancing with Indirect Draw Calls

Also called: DrawMeshInstancedIndirect, GPU Instanced Voxel Rendering via DrawMeshInstancedIndirect, Graphics.RenderMeshIndirect, Indirect GPU Instancing, Indirect Voxel Rendering, Instanced Voxel Debug Visualization, Instanced Voxel Mesh Rendering

## Overview
GPU Instancing with Indirect Draw Calls is a rendering technique that efficiently draws millions of identical meshes (voxels) by issuing a single draw command that reads instance counts and parameters from a GPU-accessible buffer, allowing the GPU to generate draw commands itself [S1, S3, unofficial]. Unlike standard GPU instancing which requires CPU-side iteration to set up per-instance data or multiple draw calls, this method batches the rendering of all instances into one or few calls, significantly reducing CPU-GPU synchronization overhead [S1, unofficial]. It rasterizes explicit cube geometry via indirect commands, contrasting with [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md) and [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md) [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A single voxel mesh (typically a cube), a buffer of instance positions/attributes (e.g., `StructuredBuffer` or `ComputeBuffer`), and an indirect argument buffer containing draw command parameters [S1, S3, unofficial].
*   **Output:** Rasterized voxels on the screen, with empty voxels clipped based on occupancy data [S3, unofficial].

**Data Structures:**
1.  **Voxel Mesh:** A cube mesh with 24 vertices (4 per face) and 36 indices (2 triangles per face, 6 faces) to ensure flat shading via unique normals per face [S3, unofficial].
2.  **Indirect Argument Buffer (`GraphicsBuffer`):**
    *   Target: `GraphicsBuffer.Target.IndirectArguments` [S1, unofficial].
    *   Layout: Array of `GraphicsBuffer.IndirectDrawIndexedArgs` structs [S1, unofficial].
    *   Struct Fields: `indexCountPerInstance` (uint) and `instanceCount` (uint) [S1, unofficial].
    *   *Note: Standard indirect args often include 5 integers [S3, unofficial]. The array passed to `SetData` is specified as `[vertex count per instance, instance count, starting vertex index, starting index index, reserved value]` [S3, unofficial].*
3.  **Instance Data Buffer:**
    *   Type: `StructuredBuffer<float4>` in shaders or `ComputeBuffer` on the CPU [S2, S3, unofficial].
    *   Layout: Each element represents a voxel [S3, unofficial]. The x, y, z components store position [S3, unofficial]. The w-component stores occupancy (1.0 for solid, 0.0 for empty) [S3, unofficial].
    *   Size: `xGridSize * yGridSize * zGridSize` elements with a stride of 16 bytes [S3, unofficial].

**Numbered Steps:**
1.  **Generate Voxel Mesh:** Create a cube mesh with 24 vertices and 36 indices to support flat shading [S3, unofficial]. Vertices are duplicated per face to ensure unique normals [S3, unofficial]. Indices define 6 triangles per face across 6 faces [S3, unofficial].
2.  **Initialize Buffers:** Create the Indirect Argument Buffer with size `commandCount` (e.g., 1 or 2) [S1, unofficial]. Create the Instance Data Buffer with size equal to the total number of potential voxel slots [S3, unofficial]. Populate the Instance Data Buffer with voxel positions and occupancy flags [S3, unofficial].
3.  **Configure Indirect Arguments:** Set `indexCountPerInstance` to the mesh's index count (36) [S1, S3, unofficial]. Set `instanceCount` to the number of active voxels or total grid size if clipping is handled in the shader [S1, S3, unofficial]. Set other fields (start vertex/index) to 0 [S3, unofficial]. Upload this data to the GPU using `commandBuf.SetData(commandData)` [S1, unofficial].
4.  **Shader Setup (Procedural Instancing):** Define a `setup()` function to retrieve per-instance data from the position buffer using `unity_InstanceID` [S3, unofficial]. Construct a translation matrix `_Matrix` from the position (x, y, z) [S3, unofficial]. Calculate a clip value `_Clip` based on the w-component: `_Clip = -1.0 + position.w` [S3, unofficial]. If w=0 (empty), `_Clip`=-1.0; if w=1 (solid), `_Clip`=0.0 [S3, unofficial]. Enable procedural instancing via `#pragma instancing_options procedural:setup` [S3, unofficial].
5.  **Vertex Shader Processing:** Retrieve instance ID and command ID using `GetIndirectInstanceID(svInstanceID)` and `GetCommandID(0)` [S1, unofficial]. If not using a position buffer but a grid, calculate 2D/3D coordinates from instance ID: `x = instanceID % _GridWidth`, `z = instanceID / _GridWidth` [S1, unofficial]. Transform vertex: `v.vertex = mul(_Matrix, v.vertex)` [S3, unofficial] or `wpos = mul(_ObjectToWorld, v.vertex + float4(gridPosition, 0))` [S1, unofficial]. Output position: `o.pos = mul(UNITY_MATRIX_VP, wpos)` [S1, unofficial].
6.  **Fragment Shader Processing:** Discard fragments where `_Clip < 0` using `clip(_Clip)` [S3, unofficial]. Assign color using a deterministic hash based on instance ID or grid position [S1, S2, unofficial].
7.  **Execute Draw Call:** Call `Graphics.DrawMeshInstancedIndirect(mesh, 0, material, bounds, argsBuffer)` [S3, unofficial] or `Graphics.RenderMeshIndirect(rp, mesh, commandBuf, commandCount)` [S1, unofficial].

**Pseudocode (CPU Setup & Draw):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_indirect_instancing(voxel_mesh, grid_dims, occupancy_buffer):
    # 1. Initialize indirect argument buffer
    command_count = 1
    command_buf = GraphicsBuffer(target=GraphicsBuffer.Target.IndirectArguments, 
                                 count=command_count, 
                                 stride=GraphicsBuffer.IndirectDrawIndexedArgs.size)
    
    # 2. Initialize instance data buffer
    total_voxels = grid_dims.x * grid_dims.y * grid_dims.z
    voxel_points_buf = ComputeBuffer(count=total_voxels, stride=16)
    
    # Populate voxel_points_buf with positions and occupancy (w=1 for solid, w=0 for empty)
    # Not given in the sources.

    # 3. Configure indirect arguments
    # Args: [indexCountPerInstance, instanceCount, startVertex, startIndex, reserved]
    args = [voxel_mesh.index_count, total_voxels, 0, 0, 0]
    command_buf.set_data(args)
    
    # 4. Execute indirect draw
    Graphics.DrawMeshInstancedIndirect(
        mesh=voxel_mesh, 
        submesh_index=0, 
        material=material, 
        bounds=mesh_colliders.bounds, 
        args_buffer=command_buf
    )
    return command_buf, voxel_points_buf

def cleanup(command_buf, voxel_points_buf):
    # 5. Cleanup to prevent memory leaks
    if command_buf is not None:
        command_buf.release()
    if voxel_points_buf is not None:
        voxel_points_buf.release()
```

**Pseudocode (Shader Logic):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```hlsl
#ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
StructuredBuffer<float4> _Positions;
float4x4 _Matrix;
half _Clip;

void setup() {
    float4 position = _Positions[unity_InstanceID];
    _Matrix = float4x4(
        1, 0, 0, position.x,
        0, 1, 0, position.y,
        0, 0, 1, position.z,
        0, 0, 0, 1
    );
    // Clip empty voxels (w=0 -> _Clip=-1 -> discard)
    _Clip = -1.0 + position.w; 
}
#endif

void vert(inout appdata_full v, out Input data) {
    UNITY_INITIALIZE_OUTPUT(Input, data);
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    v.vertex = mul(_Matrix, v.vertex);
    #endif
}

void surf(Input IN, inout SurfaceOutputStandard o) {
    clip(_Clip); // Discard if _Clip < 0
    o.Albedo = _Color.rgb;
}
```

**Formulas:**
*   **Grid Position (if not using position buffer):**
    $$x = \text{instanceID} \pmod{\text{GridWidth}}$$
    $$z = \lfloor \text{instanceID} / \text{GridWidth} \rfloor$$
    $$\text{gridPosition} = (x \cdot \text{Spacing}, 0, z \cdot \text{Spacing})$$ [S1, unofficial]
*   **Clip Value:**
    $$\text{Clip} = -1.0 + \text{position}.w$$ [S3, unofficial]
*   **Deterministic Color Hash:**
    $$n = (n \ll 13U) \oplus n$$
    $$n = n \cdot (n \cdot n \cdot 15731U + 0x789221U) + 0x1376312589U$$
    $$\text{color} = \frac{n \ \& \ 0x7fffffffU}{0x7fffffff}$$ [S2, unofficial]

**Parameters:**
*   `commandCount`: Number of indirect draw commands (typically 1) [S1, unofficial].
*   `gridWidth`, `gridHeight`, `gridDepth`: Dimensions of the voxel grid [S1, S3, unofficial].
*   `spacing`: Distance between instances (if using grid calculation) [S1, unofficial].
*   `voxelSize`: Size of the cube mesh [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Leaks:** Failing to release `GraphicsBuffer` and `ComputeBuffer` in `OnDestroy` causes memory leaks [S1, unofficial]. Use null-conditional operators (`?.`) for safety [S1, unofficial].
*   **Empty Voxels:** If the w-component of the position buffer is 0, the fragment is discarded [S3, unofficial]. If not handled, empty voxels render as solid blocks [S3, unofficial].
*   **Precision:** Using `uint` for instance IDs is necessary for large grids (e.g., 1 million instances) [S1, unofficial].
*   **Null References:** Ensure buffers are not null before calling `Release()` or `SetData()` [S1, unofficial].

## Subtopics

### [Chunk Rendering with Multidraw Indirect](3933-chunk-rendering-with-multidraw-indirect.md)
Batches chunk rendering using `glMultiDrawElementsIndirect` to reduce draw call overhead, storing chunk geometry in a large shared VBO. Choose this when rendering large, static voxel worlds with chunk-based culling.

## Operating characteristics
*   **Time Complexity:** $O(1)$ draw calls regardless of the number of instances, reducing CPU overhead from $O(N)$ to $O(1)$ [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for instance data buffers, where $N$ is the number of potential voxel slots [S3, unofficial].
*   **Performance:** Capable of rendering 1 million voxels simultaneously on integrated GPUs (e.g., Intel XE Graphics) [S1, unofficial].
*   **Strengths:** Minimal CPU overhead, scalable to millions of instances, simple implementation via standard APIs [S1, S3, unofficial].
*   **Limitations:** Requires GPU support for indirect drawing (DirectX 11 / OpenGL 4.3+) [S1, unofficial]. Clipping empty voxels in the shader can cause overdraw if not carefully managed [S3, unofficial].
*   **When to Use:** Debugging voxel occupancy, rendering large static voxel sets, or when CPU-side instancing setup is a bottleneck [S1, S3, unofficial].
*   **When Not to Use:** When dynamic per-vertex data is required that cannot be stored in buffers, or on very old hardware lacking indirect draw support.

## Background
This technique was popularized in the context of high-performance voxel engines in Unity, specifically demonstrated in "Building a High-Performance Voxel Engine in Unity: A Step-by-Step Guide Part 7: 'One Million Voxels'" by CodeComplex (2024) [S1, unofficial]. It leverages the `DrawMeshInstancedIndirect` API introduced in Unity to batch draw calls, contrasting with earlier optimization techniques like sparse voxel DAGs [S1, unofficial]. The method is also used in GPU mesh voxelizers to render voxelized meshes efficiently [S3, unofficial].

## Sources
[S1] Building a High-Performance Voxel Engine in Unity: A Step-by ... - Medium, https://medium.com/@adamy1558/building-a-high-performance-voxel-engine-in-unity-a-step-by-step-guide-part-7-one-million-ed987e894b71, fetched 2026-10-27 (unofficial)
[S2] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
[S3] GPU Mesh Voxelizer Part 3: Render tons of voxels with ..., https://bronsonzgeb.com/index.php/2021/06/05/gpu-mesh-voxelizer-part-3-render-tons-of-voxels-with-drawmeshinstancedindirect/, fetched 2026-10-29 (unofficial)
