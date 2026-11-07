# GPU Instancing with Indirect Draw Calls

Also called: DrawMeshInstancedIndirect, GPU Instanced Voxel Rendering via DrawMeshInstancedIndirect, Graphics.RenderMeshIndirect, Indirect GPU Instancing, Indirect Voxel Rendering, Instanced Voxel Debug Visualization, Instanced Voxel Mesh Rendering

## Overview
GPU Instancing with Indirect Draw Calls is a technique to render a large number of identical meshes (voxels) efficiently by using indirect draw calls [S1, unofficial]. Instead of issuing one draw call per instance, a single draw call is issued that reads instance counts and parameters from a GPU-accessible buffer, allowing the GPU to generate the draw commands itself [S1, unofficial][S3, unofficial]. It is distinguished from its siblings by decoupling the CPU-side draw call issuance from the GPU-side execution logic, enabling dynamic instance counts and reducing CPU-GPU synchronization overhead compared to standard hardware instancing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A single mesh geometry (e.g., a voxel cube), a material with procedural instancing support, and a buffer containing instance data (positions, occupancy) and indirect draw arguments.
*   **Output:** Rasterized voxels on the screen.

**Data Structures:**

1.  **Voxel Mesh:** A cube mesh with 24 vertices (4 per face) and 36 indices (2 triangles per face, 6 faces) to ensure flat shading by duplicating vertices for unique normals per face [S3, unofficial].
2.  **Instance Position Buffer:** A `StructuredBuffer<float4>` (or `ComputeBuffer` with stride 16 bytes) named `_Positions` [S3, unofficial]. Each element stores the voxel's position in the x, y, z components and an occupancy flag in the w component (1.0 for solid, 0.0 for empty) [S3, unofficial].
3.  **Indirect Arguments Buffer:** A `GraphicsBuffer` of type `IndirectArguments` [S1, unofficial]. It stores an array of `IndirectDrawIndexedArgs` structs. Each struct contains:
    *   `indexCountPerInstance` (uint): Number of indices per instance (e.g., 36).
    *   `instanceCount` (uint): Number of instances to draw.
    *   `firstVertex` (uint): Starting vertex index (usually 0).
    *   `firstIndex` (uint): Starting index index (usually 0).
    *   `reserved` (uint): Reserved value (usually 0) [S3, unofficial].

**Numbered Steps:**

1.  **Generate Voxel Mesh:** Create a cube mesh with 24 vertices and 36 indices [S3, unofficial].
    *   Vertices are defined for 6 faces, with 4 vertices per face.
    *   Indices define 2 triangles per face in clockwise order [S3, unofficial].
2.  **Initialize Buffers:**
    *   Create the `_Positions` buffer with size equal to the total number of potential voxel slots (e.g., `xGridSize * yGridSize * zGridSize`) [S3, unofficial].
    *   Create the `commandBuf` (Indirect Arguments Buffer) with size equal to `commandCount` (number of indirect commands) [S1, unofficial].
3.  **Configure Indirect Arguments:**
    *   Set the `indexCountPerInstance` to the mesh's triangle count (36) [S3, unofficial].
    *   Set the `instanceCount` to the number of active voxels [S3, unofficial].
    *   Set `firstVertex`, `firstIndex`, and `reserved` to 0 [S3, unofficial].
    *   Upload this data to the `commandBuf` using `SetData` [S1, unofficial].
4.  **Setup Shader:**
    *   Define a `setup()` function in the shader to retrieve per-instance data from `_Positions` using `unity_InstanceID` [S3, unofficial].
    *   Construct a translation matrix `_Matrix` from the position.x, y, z components [S3, unofficial].
    *   Calculate a clip value `_Clip` based on the w component: `_Clip = -1.0 + position.w` [S3, unofficial].
    *   Enable procedural instancing via `#pragma instancing_options procedural:setup` [S3, unofficial].
5.  **Execute Draw Call:**
    *   Call `Graphics.DrawMeshInstancedIndirect` (or `Graphics.RenderMeshIndirect` in Unity) passing the mesh, submesh index (0), material, bounds, and the `commandBuf` [S3, unofficial][S1, unofficial].

**Pseudocode (C# / Unity API):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```csharp
// 1. Generate Voxel Mesh
Mesh GenerateVoxelMesh(float size) {
    Vector3[] vertices = new Vector3[24];
    int[] triangles = new int[36];
    
    // Define 24 vertices for 6 faces (4 vertices per face)
    // Front face (z=0)
    vertices[0] = new Vector3(0, 0, 0);
    vertices[1] = new Vector3(size, 0, 0);
    vertices[2] = new Vector3(size, size, 0);
    vertices[3] = new Vector3(0, size, 0);
    // ... (Define remaining 20 vertices for Top, Right, Left, Back, Bottom faces) ...
    
    // Define 36 indices (2 triangles per face, 6 faces)
    // Front face
    triangles[0] = 0; triangles[1] = 2; triangles[2] = 1;
    triangles[3] = 0; triangles[4] = 3; triangles[5] = 2;
    // ... (Define remaining indices for other faces) ...
    
    Mesh mesh = new Mesh();
    mesh.SetVertices(vertices);
    mesh.SetTriangles(triangles, 0);
    mesh.RecalculateBounds();
    mesh.RecalculateNormals();
    return mesh;
}

// 2. Initialize Buffers
GraphicsBuffer commandBuf;
GraphicsBuffer.IndirectDrawIndexedArgs[] commandData;
const int commandCount = 2; // Example count
int totalInstances;

void Start() {
    commandBuf = new GraphicsBuffer(GraphicsBuffer.Target.IndirectArguments, commandCount, GraphicsBuffer.IndirectDrawIndexedArgs.size);
    commandData = new GraphicsBuffer.IndirectDrawIndexedArgs[commandCount];
    totalInstances = gridWidth * gridHeight; // Or xGridSize * yGridSize * zGridSize
}

// 3. Configure Indirect Arguments
void Update() {
    // Set up the indirect argument data
    commandData[0].indexCountPerInstance = mesh.GetIndexCount(0); // 36
    commandData[0].instanceCount = (uint)totalInstances;
    commandData[0].firstVertex = 0;
    commandData[0].firstIndex = 0;
    commandData[0].reserved = 0;
    
    // Upload to GPU
    commandBuf.SetData(commandData);
    
    // Set instance position buffer in material
    material.SetBuffer("_Positions", _voxelPointsBuffer);
    
    // 5. Execute Draw Call
    Graphics.DrawMeshInstancedIndirect(mesh, 0, material, bounds, commandBuf);
}

// 4. Cleanup
void OnDestroy() {
    commandBuf?.Release(); // Null-conditional operator to prevent null reference
    commandBuf = null;
}
```

**Shader Pseudocode (HLSL/Unity ShaderLab):**

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
    _Clip = -1.0 + position.w;
}
#endif

void vert(inout appdata_full v, out Input data) {
    UNITY_INITIALIZE_OUTPUT(Input, data);
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    v.vertex = mul(_Matrix, v.vertex);
    #endif
}

void surf (Input IN, inout SurfaceOutputStandard o) {
    clip(_Clip); // Discard if _Clip < 0 (empty voxel)
    o.Albedo = _Color.rgb;
}
```

**Edge Cases and Pitfalls:**
*   **Memory Leak:** If the `GraphicsBuffer` is not released in `OnDestroy`, it causes memory leaks [S1, unofficial].
*   **Null Reference:** Use null-conditional operators (`?.`) when releasing buffers to avoid null reference exceptions [S1, unofficial].
*   **Empty Voxel Clipping:** If the w-component of the position buffer is 0 (empty voxel), the clip value becomes -1.0, causing the fragment to be discarded. If w is 1 (solid), the clip value becomes 0.0, keeping the fragment [S3, unofficial].
*   **Precision:** Ensure the instance ID calculation matches the buffer layout to avoid accessing out-of-bounds memory [S3, unofficial].

## Subtopics

### [Chunk Rendering with Multidraw Indirect](3933-chunk-rendering-with-multidraw-indirect.md)
Aggregates mesh data from multiple world chunks into a single large buffer and encodes chunk identity into the instance ID or base vertex, rather than relying on per-instance vertex attributes for position. Choose this when rendering large voxel worlds with chunk-based culling.

## Operating characteristics
*   **Time Complexity:** O(1) draw calls regardless of the number of instances, as all instances are drawn in a single indirect call [S1, unofficial].
*   **Memory Complexity:** O(N) for instance data, where N is the number of potential voxel slots [S3, unofficial].
*   **Measured Performance:** Rendering 1 million voxels simultaneously on a laptop with Intel XE Graphics (integrated GPU) is feasible [S1, unofficial].
*   **Strengths:** Reduces CPU overhead by batching draw calls; allows dynamic instance counts without CPU-side loop overhead [S1, unofficial].
*   **Limitations:** Requires GPU-accessible buffers and shader support for procedural instancing; may have higher latency for updating instance data compared to static instancing [S3, unofficial].
*   **When to Use:** When rendering millions of identical objects (voxels) with dynamic occupancy or position updates [S3, unofficial].
*   **When Not to Use:** When the number of instances is small, or when the overhead of buffer management outweighs the draw call savings [S1, unofficial].

## Background
This technique was popularized in the context of high-performance voxel engines in Unity, specifically in guides detailing the rendering of "One Million Voxels" [S1, unofficial]. It leverages the `DrawMeshInstancedIndirect` API introduced in Unity to handle large-scale voxel rendering efficiently [S3, unofficial]. The method contrasts with previous optimization techniques like sparse voxel DAGs by using indirect draw calls to batch rendering [S1, unofficial].

## Sources
[S1] Building a High-Performance Voxel Engine in Unity: A Step-by ... - Medium, https://medium.com/@adamy1558/building-a-high-performance-voxel-engine-in-unity-a-step-by-step-guide-part-7-one-million-ed987e894b71, fetched 2026-10-27 (unofficial)
[S2] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
[S3] GPU Mesh Voxelizer Part 3: Render tons of voxels with ..., https://bronsonzgeb.com/index.php/2021/06/05/gpu-mesh-voxelizer-part-3-render-tons-of-voxels-with-drawmeshinstancedindirect/, fetched 2026-10-29 (unofficial)
