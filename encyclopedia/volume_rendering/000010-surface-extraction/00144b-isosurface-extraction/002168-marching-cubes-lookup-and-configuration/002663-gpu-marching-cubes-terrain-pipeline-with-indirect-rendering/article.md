# GPU Marching Cubes Terrain Pipeline with Indirect Rendering

Also called: GPU Marching Cubes with Indirect Rendering, Indirect Marching Cubes, Keijiro's GPU Marching Cubes (modified), Marching Cubes/MeshBuilderIndirect, TerrainMeshBuilderIndirect

## Overview
This method extracts an implicit terrain surface from voxel data on the GPU using Marching Cubes, managing multiple Level of Detail (LOD) levels and rendering the generated triangles via indirect draw calls [S1, unofficial]. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by replacing CPU-GPU synchronization stalls with GPU-generated draw arguments, allowing the GPU to render the mesh directly without readback [S1, unofficial]. Unlike [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md) which may require CPU-side buffer management or synchronization, this approach keeps the RenderThread time low (~1.5ms) by keeping all draw argument generation on the GPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Voxel data buffer (scalar field), precomputed `triangleTable` (256 `ulong` entries), voxel resolution (`int3`), voxel size (`float`), isovalue (`float`, typically 0.0), chunk center in world space (`float3`), and LOD configuration (`TerrainLod` list).
*   **Outputs:** Triangle meshes rendered via indirect draw calls. The output is managed in per-LOD `AppendStructuredBuffer`s for vertices and `ComputeBuffer`s for indirect arguments.

**Data Structures:**
1.  **`Triangle` Struct:** Contains three `VertexGPU` structs (`v0`, `v1`, `v2`).
    *   `VertexGPU` contains: `float3 posWS` (world position), `float3 nrmWS` (world normal), `float2 uv`.
    *   Total size: 96 bytes per triangle (32 bytes per vertex) [S1, unofficial].
2.  **`TerrainLod` Struct:**
    *   `LOD`: int (1-based index).
    *   `VoxelResolution`: `int3`.
    *   `chunkRadius`: int.
    *   LODs must be ordered highest-to-lowest resolution [S1, unofficial].
3.  **`IndirectArgBuffers`:** 5 `uint` values per LOD.
    *   Index 0: Total vertex count (updated dynamically).
    *   Index 1: Instance count (number of triangles).
    *   Index 2: Start vertex (0).
    *   Index 3: Start instance (0).
    *   Index 4: Unused (0) [S1, unofficial].
4.  **`triangleTable`:** `ComputeBuffer` of 256 `ulong` values, preloaded with the standard Marching Cubes edge configuration table [S1, unofficial].

**Algorithm Steps:**

1.  **Buffer Allocation:**
    For each LOD level $i$, calculate the active chunk count using a flood-fill or spatial query (`ManhattanFloodFillCount3D`).
    *   For LOD 0, use the full active chunk count.
    *   For $i > 0$, subtract the active chunk count of the previous LOD level.
    *   Allocate `VertexBuffers` as `AppendStructuredBuffer` sized to:
        $$ \text{Size} = \text{voxelRes}_x \times \text{voxelRes}_y \times \text{voxelRes}_z \times \text{activeChunks} $$
    *   Allocate `IndirectArgBuffers` as 5 `uint` values [S1, unofficial].

2.  **Initialize Indirect Arguments:**
    Set the initial values in `IndirectArgBuffers` to `{3, 1, 0, 0, 0}`:
    *   `vertexCountPerInstance` = 3 (vertices per triangle).
    *   `instanceCount` = 1 (placeholder, updated later).
    *   `startVertex` = 0.
    *   `startInstance` = 0.
    *   `unused` = 0 [S1, unofficial].

3.  **Dispatch Marching Cubes Compute Kernel:**
    *   Bind `Voxels` buffer, `TriangleTable` buffer, and `g_Tris` (the `VertexBuffer` for the current LOD) to the compute shader.
    *   Set parameters:
        *   `Dims`: Voxel resolution $(N_x, N_y, N_z)$.
        *   `Scale`: Voxel size.
        *   `Isovalue`: 0.0.
        *   `HalfVoxelOffset`: $\text{Scale} \times -0.5$ (applied to all axes to center the grid) [S1, unofficial].
        *   `chunkCenter`: Chunk center in world space.
    *   Dispatch the kernel with group counts $\lceil N_x/4 \rceil \times \lceil N_y/4 \rceil \times \lceil N_z/4 \rceil$, implying a $4 \times 4 \times 4$ thread group [S1, unofficial].
    *   *Kernel Logic (per thread/voxel):*
        1.  Sample scalar values at the 8 corners of the voxel.
        2.  Compute the 8-bit case index by comparing values against the isovalue.
        3.  Look up `triangleTable[caseIndex]` to determine active edges.
        4.  For each active edge, compute intersection point $\mathbf{p}$ via linear interpolation:
            $$ \mathbf{p} = \mathbf{p}_1 + \frac{\tau - V_1}{V_2 - V_1} (\mathbf{p}_2 - \mathbf{p}_1) $$
            where $V_1, V_2$ are corner values and $\mathbf{p}_1, \mathbf{p}_2$ are corner positions [S1, Parent, unofficial].
        5.  Compute normals via gradient interpolation or finite differences [S1, Parent, unofficial].
        6.  Emit triangles to the `AppendStructuredBuffer` `g_Tris` using atomic operations or hardware append buffer support [S1, Parent, unofficial].

4.  **Update Indirect Arguments:**
    *   Copy the triangle append counter from the `VertexBuffer` to the `IndirectArgBuffer` at byte offset 0.
    *   Dispatch a helper compute shader (`indirectArgumentSetter`) to update the arguments:
        *   `args[0]` (vertex count) = `triangleCount` $\times 3$.
        *   `args[1]` (instance count) = `triangleCount` $\times 1$.
        *   `args[2..4]` remain 0 [S1, unofficial].

5.  **Enqueue Indirect Draw Call:**
    *   Create an `IndirectDrawCall` entry containing:
        *   `tris`: The `VertexBuffer` (bound as `_Tris` in the render pass).
        *   `args`: The `IndirectArgBuffer`.
        *   `material`: Terrain material.
        *   `trs`: Identity transform matrix (or chunk-specific transform if applied in shader) [S1, unofficial].
    *   Register the entry with `GpuIndirectDrawRegistry.Add()`.

6.  **Reset Append Counter:**
    *   Call `vertexBuffer.SetCounterValue(0)` to clear the triangle append buffer for the next frame [S1, unofficial].

**Vertex Shader:**
*   Fetches the corner vertex from the input buffer: if corner index is 0, return `t.v0`; if 1, return `t.v1`; else return `t.v2`.
*   Transforms world-space position to clip space: `TransformWorldToHClip(o.posWS)` [S1, unofficial].

**Fragment Shader:**
*   Normalizes normal vector `N`.
*   If $\mathbf{N} \cdot \mathbf{V} < 0$ (where $\mathbf{V}$ is view direction), flip `N` to ensure it points towards the camera.
*   Calculates Lambertian lighting using a fixed light direction `float3(1,1,0)`.
*   Returns a value interpolated between 0.25 and 1.0 based on the dot product [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the GPU Pipeline Management (CPU-side setup, GPU-side execution)

class TerrainMeshBuilderIndirect:
    def __init__(self, voxel_volume_size, marching_cubes_shader, indirect_arg_adjuster_shader, terrain_material, lods):
        self.lods = lods
        self.terrain_material = terrain_material
        self.marching_cubes_shader = marching_cubes_shader
        self.indirect_arg_adjuster_shader = indirect_arg_adjuster_shader
        
        # Allocate per-LOD buffers
        self.vertex_buffers = []
        self.indirect_arg_buffers = []
        
        for lod in self.lods:
            # Calculate active chunks for this LOD
            # Note: ManhattanFloodFillCount3D is a hypothetical helper to count active chunks
            active_chunks = ManhattanFloodFillCount3D(lod.voxel_resolution)
            
            # For LOD > 0, subtract previous LOD's count (simplified logic)
            if lod.lod > 0:
                # In practice, this requires tracking active chunks per LOD level
                pass 
            
            # Buffer size: voxel volume * active chunks
            buffer_size = lod.voxel_resolution.x * lod.voxel_resolution.y * lod.voxel_resolution.z * active_chunks
            
            # VertexBuffer: AppendStructuredBuffer for triangles
            # Each triangle is 96 bytes (3 vertices * 32 bytes)
            vb = CreateAppendStructuredBuffer(buffer_size * 96)
            self.vertex_buffers.append(vb)
            
            # IndirectArgBuffer: 5 uints
            # Initial: {3, 1, 0, 0, 0}
            iab = CreateComputeBuffer(5, uint)
            iab.set_data([3, 1, 0, 0, 0])
            self.indirect_arg_buffers.append(iab)

    def calculate_mesh(self, chunk_index, voxel_buffer, voxel_size, voxel_resolution, chunk_center_ws, lod):
        # Get buffers for this LOD
        vertex_buffer = self.vertex_buffers[lod.lod - 1]
        
        # Reset append counter for this frame
        vertex_buffer.set_counter_value(0)
        
        # Dispatch Marching Cubes Compute Shader
        self.marching_cubes_shader.set_buffer(0, "g_Tris", vertex_buffer)
        self.marching_cubes_shader.set_buffer(0, "Voxels", voxel_buffer)
        self.marching_cubes_shader.set_buffer(0, "TriangleTable", self.triangle_table_buffer)
        
        self.marching_cubes_shader.set_ints("Dims", voxel_resolution)
        self.marching_cubes_shader.set_float("Scale", voxel_size)
        self.marching_cubes_shader.set_float("Isovalue", 0.0)
        
        half_voxel_offset = voxel_size * -0.5
        self.marching_cubes_shader.set_floats("HalfVoxelOffset", half_voxel_offset, half_voxel_offset, half_voxel_offset)
        self.marching_cubes_shader.set_floats("chunkCenter", chunk_center_ws.x, chunk_center_ws.y, chunk_center_ws.z)
        
        # Dispatch with 4x4x4 thread groups
        group_count_x = ceil(voxel_resolution.x / 4)
        group_count_y = ceil(voxel_resolution.y / 4)
        group_count_z = ceil(voxel_resolution.z / 4)
        
        self.marching_cubes_shader.dispatch(0, group_count_x, group_count_y, group_count_z)
        
        # Update Indirect Arguments
        indirect_arg_buffer = self.indirect_arg_buffers[lod.lod - 1]
        
        # Copy triangle count from vertex buffer to indirect args
        # This requires a copy operation or a compute shader to read the append counter
        self.indirect_arg_adjuster_shader.set_buffer(0, "IndirectArgs", indirect_arg_buffer)
        self.indirect_arg_adjuster_shader.set_buffer(0, "TriangleCount", vertex_buffer.counter_buffer)
        
        # Dispatch helper to update args[0] = count * 3, args[1] = count * 1
        self.indirect_arg_adjuster_shader.dispatch(0, 1, 1, 1)
        
        # Enqueue Indirect Draw Call
        indirect_draw_call = {
            "tris": vertex_buffer,
            "args": indirect_arg_buffer,
            "material": self.terrain_material,
            "trs": IdentityMatrix()
        }
        
        GpuIndirectDrawRegistry.add(indirect_draw_call)
        
        # Reset counter for next frame
        vertex_buffer.set_counter_value(0)

# Vertex Shader Pseudocode (HLSL-like)
struct VertexInput {
    uint vertex_id;
};

struct VertexOutput {
    float4 posHClip : SV_POSITION;
    float3 posWS : TEXCOORD0;
    float3 nrmWS : TEXCOORD1;
    float2 uv : TEXCOORD2;
};

VertexOutput VS(VertexInput input) {
    // Fetch triangle from AppendStructuredBuffer
    Triangle t = g_Tris[input.vertex_id / 3];
    uint corner_index = input.vertex_id % 3;
    
    VertexGPU v;
    if (corner_index == 0) v = t.v0;
    else if (corner_index == 1) v = t.v1;
    else v = t.v2;
    
    VertexOutput output;
    output.posHClip = TransformWorldToHClip(v.posWS);
    output.posWS = v.posWS;
    output.nrmWS = v.nrmWS;
    output.uv = v.uv;
    
    return output;
}

// Fragment Shader Pseudocode
float4 PS(VertexOutput input) : SV_TARGET {
    float3 N = normalize(input.nrmWS);
    float3 V = normalize(CameraPosition - input.posWS);
    
    // Flip normal if pointing away from camera
    if (dot(N, V) < 0.0) {
        N = -N;
    }
    
    float3 L = normalize(float3(1, 1, 0));
    float diff = max(dot(N, L), 0.0);
    
    // Interpolate between 0.25 and 1.0
    float lighting = lerp(0.25, 1.0, diff);
    
    return float4(lighting, lighting, lighting, 1.0);
}
```

## Subtopics

### [Marching Cubes Chunk Generation Pipeline](3925-marching-cubes-chunk-generation-pipeline.md)
This method is a procedural terrain generation pipeline that divides a 3D volume into overlapping chunks, populates them with noise-derived node values, and prepares chunk meshes for Marching Cubes surface extraction. It builds on [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md) by handling the CPU-side data preparation and chunk management required for the GPU-accelerated extraction. Unlike single-volume extraction, this approach manages spatial locality and boundary continuity through overlapping node grids, ensuring seamless transitions between adjacent chunks.

## Operating characteristics
*   **Time Complexity:** The compute kernel runs in $O(V)$ where $V$ is the number of voxels in the chunk. The indirect argument update and draw call enqueue are $O(1)$ per LOD level.
*   **Memory Complexity:** Memory scales linearly with voxel volume and active chunks per LOD: $O(\text{VoxelResolution}_x \times \text{VoxelResolution}_y \times \text{VoxelResolution}_z \times \text{activeChunks})$. Each triangle consumes 96 bytes in the append buffer [S1, unofficial].
*   **Performance:** Keeps RenderThread time low (~1.5ms) by avoiding CPU-GPU synchronization stalls associated with readback [S1, unofficial].
*   **Strengths:** No CPU mesh copy required; efficient handling of multiple LOD levels; minimal CPU overhead.
*   **Limitations:** Requires custom physics/collision solutions since the mesh is not available on the CPU [S1, unofficial]. Voxel resolution must be odd; even resolutions trigger a validation error [S1, unofficial]. Chunk dimensions must be cube-like ($x == y == z$); non-cubic chunks trigger a validation error [S1, unofficial].
*   **When to use:** Real-time terrain rendering in Unity or similar engines where CPU-GPU synchronization is a bottleneck and physics can be handled via voxel-based collision or other GPU-friendly methods.
*   **When not to use:** When CPU-side mesh access is required for physics or collision detection.

## Background
This method is implemented in Unity terrain rendering pipelines using ComputeShader for surface extraction and `GpuIndirectDrawRegistry` for batched rendering [S1, unofficial]. It is a modification of Keijiro's GPU Marching Cubes, adapted to use indirect rendering to avoid CPU readback [S1, unofficial]. The shader target version is 4.5 [S1, unofficial].

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
