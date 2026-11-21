# Geometry Shader Voxel Rendering

Also called: GS Voxel Spawning, Voxel-to-Geometry Shader

## Overview
A technique where each input vertex represents a voxel, and a geometry shader spawns individual cubic geometry for each voxel [S1, S2, unofficial]. It differs from [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md) by typically processing sparse voxel data where vertices explicitly denote occupied cells, rather than expanding a dense grid or marching cubes approach. It is chosen when immediate mode geometry generation is preferred over ray-marching or ray-casting, particularly for simple voxel terrains where view distance is limited.

## How it works
**Inputs and Outputs:**
- Input: A stream of vertices where each vertex represents a single voxel in 3D space. The vertex attributes must encode the voxel's position and material/type data.
- Output: A stream of triangles forming the visible faces of the voxel cubes.

**Data Structures:**
Vertices are packed with bitflags to maximize space efficiency [S1, S2, unofficial].
- **Material Type:** Stored in the 5 least significant bits (LSB) of a byte/integer field. This supports $2^5 = 32$ distinct material types [S1, S2, unofficial].
- **Other Data:** Stored in the 3 most significant bits (MSB) of the same byte/integer field. This supports $2^3 = 8$ distinct values for auxiliary data (e.g., lighting, flags) without allocating a new byte [S1, S2, unofficial].
- **Position:** Implicitly derived from the vertex index or stored in separate attributes (not specified in sources, but implied by "each vertex represents a voxel").

**Steps:**
1. **Pack Data:** Encode voxel material type and auxiliary data into a single integer/byte using bitfields as described above [S1, S2, unofficial].
2. **Upload:** Send the packed vertex data to the GPU [S1, S2, unofficial].
3. **Geometry Shader Execution:** For each input vertex:
   a. Extract the material type and other data using bitwise operations [S1, S2, unofficial].
   b. Determine the 3D position of the voxel based on the vertex input [G].
   c. Generate the geometry for the voxel cube. This typically involves generating 6 faces (12 triangles) or only visible faces if culling is implemented in the GS [G].
   d. Assign the extracted material type to the output vertices.
   e. Emit the primitives.
4. **Rasterization:** The generated cubes are rasterized by the fixed-function pipeline.

**Parameters:**
- **View Distance:** A critical parameter. The technique generates a large amount of geometry proportional to the number of visible voxels. View distance must be limited to prevent performance collapse [S1, S2, unofficial].
- **Voxel Size:** Smaller voxel sizes increase the vertex count and geometry generation load [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Bitwise Operator Support:** Extracting packed bit data on the GPU requires newer versions of GLSL and HLSL that support bitwise operators. Older shader models may not support this, requiring alternative unpacking methods [S1, S2, unofficial].
- **Geometry Overload:** The brute-force generation of geometry can spam the pipeline, leading to low frame rates if view distance or voxel density is too high [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Input
struct VoxelVertex {
    int packed_data; # Contains material (5 LSB) and other data (3 MSB)
    vec3 position;   # Voxel center or corner, depending on convention
};

# Geometry Shader
layout(triangles) in;
layout(triangle_strip, max_vertices = 24) out; # 6 faces * 4 vertices, or optimized for visible faces

void main() {
    int packed = gl_in[0].gl_PackedData;
    
    // Extract material type (5 LSB)
    int material_type = packed & 0x1F; 
    
    // Extract other data (3 MSB)
    int other_data = (packed >> 5) & 0x07;
    
    vec3 voxel_pos = gl_in[0].gl_Position.xyz;
    float voxel_size = 1.0; // Assumed unit size or passed as uniform
    
    // Generate cube geometry
    // Note: Full cube generation is 6 faces. 
    // Optimization: Only generate visible faces based on neighbor checks (not detailed in sources)
    
    // Example: Generate one face (front) for simplicity
    // In practice, all 6 faces or cullable faces should be generated
    
    vec3 offset = vec3(0.0);
    // ... Generate vertices for the cube faces using voxel_pos and voxel_size ...
    // Assign gl_MaterialIndex or similar attribute based on material_type
    // Emit vertices
}
```

## Subtopics

### [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md)
Expands a single vertex into a cube, often used for dense grids or particle systems. Choose when you have a dense grid and want to avoid per-voxel vertex overhead.

### [Velocity Visualization via Geometry Shader Extrusion](1581-velocity-visualization-via-geometry-shader-extrusion.md)
Extrudes geometry based on velocity vectors. Choose for flow visualization, not voxel terrain.

### [Geometry Shader Triangle Slicing (Z-Pass)](1681-geometry-shader-triangle-slicing-z-pass.md)
Slices triangles for depth-based effects. Choose for specific depth effects, not voxel generation.

### [Geometry Shader (Quads)](4369-geometry-shader-quads.md)
General quad generation. Choose for billboards or planes, not voxels.

### [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md)
Renders particles as points or sprites. Choose for particles, not structured voxels.

### [Recursive Geometry Shader Beam Generation](8395-recursive-geometry-beam-generation.md)
Recursively generates beams. Choose for procedural beams, not voxels.

## Operating characteristics
- **Time Complexity:** $O(V)$, where $V$ is the number of visible voxels. Each voxel generates a constant amount of geometry (6 faces).
- **Memory Complexity:** $O(V)$ for vertex buffers and generated geometry.
- **Performance:** Highly dependent on view distance and voxel size. Can generate excessive geometry, leading to poor performance if not limited [S1, S2, unofficial].
- **Strengths:** Simple implementation, immediate feedback, easy to integrate with existing geometry shaders.
- **Limitations:** High geometry overhead, limited view distance, requires bitwise support in shaders [S1, S2, unofficial].
- **When to Use:** Small-scale voxel terrains, educational purposes, or when view distance is strictly limited.
- **When Not to Use:** Large-scale open worlds, high-resolution voxels, or when performance is critical.

## Background
Originated from game development discussions on rendering voxel terrains, specifically addressing the challenge of rendering different materials efficiently [S1, S2, unofficial]. The technique is often compared to 3D spatial subdivision methods like Run-Length Encoding (RLE) for compression [S1, S2, unofficial].

## Sources
[S1] Rendering different materials in a voxel terrain, https://gamedev.stackexchange.com/questions/45022/rendering-different-materials-in-a-voxel-terrain, fetched 2026-10-26 (unofficial)
[S2] Rendering different materials in a voxel terrain, https://gamedev.stackexchange.com/questions/45022, fetched 2026-10-28 (unofficial)
