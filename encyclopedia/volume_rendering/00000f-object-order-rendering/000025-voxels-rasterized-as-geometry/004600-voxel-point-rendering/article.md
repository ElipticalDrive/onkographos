# Voxel Point Rendering

Also called: Point-based Voxel Rendering, Solid/Non-solid Point Voxelization

## Overview
Voxel Point Rendering is a technique that represents occupied voxels as geometric points (sprites) rather than cubes or quads, submitting them to the graphics pipeline for rasterization [S1, unofficial]. It solves the problem of visualizing volumetric data with minimal geometry overhead by decoupling the voxel's logical existence from its physical mesh representation [S1, unofficial]. Unlike sibling methods that generate explicit cube faces or complex proxy geometry, this method relies on the fragment shader to color points based on a binary or scalar property (e.g., solid vs. non-solid), often utilizing instancing for performance [S1, unofficial]. It is distinct from [Binary Volume Rendering (Opacity Thresholding)](1015-binary-volume-rendering.md) which typically uses ray-marching or texture-based opacity, and from [Voxel Billboard Rendering](2554-voxel-billboard-rendering.md) which generates oriented quads.

## How it works
**Inputs and Outputs:**
*   **Input:** A structured buffer of voxel data containing position and a solid/occupancy flag.
*   **Input:** A point sprite primitive topology.
*   **Output:** A 2D image where voxels are rendered as colored points.

**Data Structures:**
The core data structure is a `StructuredBuffer` (or equivalent GPU buffer) named `_Voxels` [S1, unofficial]. Each element represents a voxel in the grid and contains:
1.  `position`: A 3D vector (`float3`) representing the world-space center of the voxel [S1, unofficial].
2.  `isSolid`: A scalar value (`float`) indicating occupancy. Typically `1.0` for solid/colliding voxels and `0.0` for empty/non-solid voxels [S1, unofficial].

The grid dimensions are defined by `_GridWidth`, `_GridHeight`, and `_GridDepth` [S1, unofficial]. The spatial bounds are defined by `_BoundsMin` (a `float3`) and `_CellHalfSize` (a `float`) [S1, unofficial].

**Algorithm Steps:**

1.  **Voxelization (Pre-processing):**
    For each voxel index $(i, j, k)$ within the grid bounds:
    a. Calculate the world-space center position $P_{world}$:
    $$
    P_{world} = \begin{bmatrix} i \cdot 2 \cdot \text{CellHalfSize} + \text{CellHalfSize} + \text{BoundsMin}_x \\ j \cdot 2 \cdot \text{CellHalfSize} + \text{CellHalfSize} + \text{BoundsMin}_y \\ k \cdot 2 \cdot \text{CellHalfSize} + \text{CellHalfSize} + \text{BoundsMin}_z \end{bmatrix}
    $$
    b. Construct an Axis-Aligned Bounding Box (AABB) centered at $P_{world}$ with extents equal to `_CellHalfSize` [S1, unofficial].
    c. Perform a collision test against the source geometry (e.g., triangles) to determine if the voxel is occupied [S1, unofficial].
    d. Store the result in `_Voxels[i + j \cdot \text{GridWidth} + k \cdot \text{GridWidth} \cdot \text{GridHeight}]`:
       - `position` = $P_{world}$
       - `isSolid` = $1.0$ if collision detected, $0.0$ otherwise [S1, unofficial].

2.  **Rendering (Vertex Shader):**
    For each instance (voxel):
    a. Retrieve `position` and `isSolid` from `_Voxels[instance_id]` [S1, unofficial].
    b. Transform `position` to clip space using the Model-View-Projection matrix (e.g., `UnityWorldToClipPos`) [S1, unofficial].
    c. Interpolate the output color $C_{out}$:
    $$
    C_{out} = \text{lerp}(\text{_Color}, \text{_CollisionColor}, \text{isSolid})
    $$
    where `_Color` is the color for non-solid voxels and `_CollisionColor` is the color for solid voxels [S1, unofficial].
    d. Output the clip-space position and $C_{out}$ [S1, unofficial].

3.  **Rendering (Fragment Shader):**
    a. Return $C_{out}$ as the final fragment color [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing: GPU Compute Shader
struct Voxel {
    float3 position;
    float isSolid;
}

StructuredBuffer<Voxel> _Voxels;
float3 _BoundsMin;
float _CellHalfSize;
int _GridWidth, _GridHeight, _GridDepth;

[numthreads(4, 4, 4)]
void FillVolume(uint3 id : SV_DispatchThreadID) {
    if (id.x >= _GridWidth || id.y >= _GridHeight || id.z >= _GridDepth)
        return;

    float cellSize = _CellHalfSize * 2.0;
    
    // Calculate world center
    float3 centerPos = float3(
        id.x * cellSize + _CellHalfSize + _BoundsMin.x,
        id.y * cellSize + _CellHalfSize + _BoundsMin.y,
        id.z * cellSize + _CellHalfSize + _BoundsMin.z
    );

    // Define AABB for collision test
    float3 aabbCenter = mul(_WorldToLocalMatrix, float4(centerPos, 1.0)).xyz;
    float3 aabbExtents = float3(_CellHalfSize, _CellHalfSize, _CellHalfSize);

    // Collision detection against source mesh triangles
    bool intersects = false;
    for (int i = 0; i < _TriangleCount; i += 3) {
        Triangle tri;
        tri.a = _MeshVertices[_MeshTriangleIndices[i]];
        tri.b = _MeshVertices[_MeshTriangleIndices[i + 1]];
        tri.c = _MeshVertices[_MeshTriangleIndices[i + 2]];
        
        if (IntersectsTriangleAABB(tri, aabbCenter, aabbExtents)) {
            intersects = true;
            break;
        }
    }

    // Store voxel data
    int voxelIndex = id.x + _GridWidth * (id.y + _GridHeight * id.z);
    _Voxels[voxelIndex].position = centerPos;
    _Voxels[voxelIndex].isSolid = intersects ? 1.0 : 0.0;
}

# Rendering: Vertex Shader
struct VertexOutput {
    float4 clipPos : SV_POSITION;
    float3 color : COLOR;
};

cbuffer MaterialParams {
    float3 _Color;
    float3 _CollisionColor;
};

VertexOutput VS(uint vertexID : SV_VertexID, uint instanceID : SV_InstanceID) {
    Voxel v = _Voxels[instanceID];
    
    // Transform to clip space
    float4 clipPos = UnityWorldToClipPos(v.position);
    
    // Interpolate color based on solidity
    float3 color = lerp(_Color, _CollisionColor, v.isSolid);
    
    VertexOutput output;
    output.clipPos = clipPos;
    output.color = color;
    return output;
}

# Rendering: Fragment Shader
float3 PS(VertexOutput input) : SV_TARGET {
    return input.color;
}
```

**Parameters:**
*   `_Color`: `float3`, color for non-solid (empty) voxels.
*   `_CollisionColor`: `float3`, color for solid (occupied) voxels [S1, unofficial].
*   `_CellHalfSize`: `float`, half the width of a voxel in world units [S1, unofficial].
*   `_BoundsMin`: `float3`, minimum corner of the voxel grid in world space [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in world-space calculations may cause gaps or overlaps if `_CellHalfSize` is very small or the grid is very large [G].
*   **Visibility:** Since points have no depth buffer interaction in the traditional sense of face culling, overlapping points may require depth testing or sorting if transparency is introduced [G].
*   **Empty Voxels:** The method renders all voxels in the buffer. If the grid is sparse, rendering empty voxels (where `isSolid == 0`) may be wasteful unless filtered by the shader or culling logic [S1, unofficial].

## Subtopics

### Voxel Module Initialization and Rendering
A variant that distinguishes between point-based and surface-based rendering modes within the same framework. It checks if the grid is a `PointDataGrid` [S2, unofficial]. If true, it enables `GL_POINT_SMOOTH` and `GL_VERTEX_PROGRAM_POINT_SIZE`, renders the interior buffer using a flat shader, and disables point smooth states [S2, unofficial]. If false, it renders interior and vector buffers with a flat shader, followed by the surface buffer with a surface shader [S2, unofficial]. This allows switching between point visualization and surface visualization without changing the core data structure [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Pre-processing is $O(N)$ where $N$ is the number of voxels, dominated by collision tests against source geometry. Rendering is $O(V)$ where $V$ is the number of voxels, as each voxel is a single draw call instance or point.
*   **Memory Complexity:** $O(N)$ for the `_Voxels` buffer.
*   **Performance:** Highly efficient for sparse data or when only the occupancy status is needed, as it avoids generating 6 faces per voxel. Performance degrades if the grid resolution is extremely high and all voxels are solid, as the GPU must process millions of points.
*   **Strengths:** Minimal geometry overhead; simple shader implementation; easy to integrate with GPU-based voxelization [S1, unofficial].
*   **Limitations:** No inherent surface normal information; difficult to apply complex lighting models without additional data; visual quality depends on point size and anti-aliasing settings [S1, unofficial].

## Background
This method is commonly used in real-time visualization tools and game engines (e.g., Unity) for debugging voxelized meshes or visualizing collision volumes [S1, unofficial]. It is also found in scientific visualization libraries like OpenVDB for quick previewing of point data grids [S2, unofficial]. The approach leverages the graphics pipeline's ability to handle large numbers of points efficiently via instancing.

## Sources
[S1] bzgeb/UnityGPUMeshVoxelizerPart5, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart5, fetched 2026-10-26 (unofficial)
[S2] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
