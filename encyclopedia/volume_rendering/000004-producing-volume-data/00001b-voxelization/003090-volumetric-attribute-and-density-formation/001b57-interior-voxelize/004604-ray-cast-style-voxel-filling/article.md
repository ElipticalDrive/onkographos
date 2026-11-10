# Ray-Cast Style Voxel Filling

Also called: FillVolume, Front-to-Back Scanline Fill, Z-Direction Interior Fill

## Overview
Ray-Cast Style Voxel Filling determines the interior of a watertight mesh by casting rays along the Z-axis through empty voxels and marking them as solid if they intersect an outward-facing triangle [S1, unofficial][S2]. It solves the problem of distinguishing interior from exterior voxels in a pre-computed surface grid by leveraging geometric intersection tests rather than topological connectivity [S1, unofficial][S2]. Unlike [Solid Voxelization](1695-solid-voxelization.md), which uses flood-fill or two-level grids to handle complex topologies robustly, this method relies on linear Z-axis scanning and is sensitive to mesh orientation and self-intersections [S1, unofficial][S3, unofficial]. It differs from [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) by using deterministic ray-triangle intersection tests on specific grid cells rather than probabilistic sampling [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid where surface voxels are already classified as `FrontFace`, `BackFace`, or `Empty` based on their normal relative to the +Z direction [S2]. A mesh defined by vertex positions and triangle indices [S1, unofficial].
*   **Output:** The same voxel grid, with interior voxels marked as `Solid` (or `fill = 1`) [S1, unofficial][S2].

**Data Structures:**
*   **Voxel Grid:** A 3D array or structured buffer. Each voxel contains flags for `isSolid`, `isFrontFace`, `isBackFace`, and `fill` [S1, unofficial][S2].
*   **Mesh Data:**
    *   `_MeshVertices`: `StructuredBuffer<float3>` containing vertex positions [S1, unofficial].
    *   `_MeshTriangleIndices`: `StructuredBuffer<int>` containing triangle vertex indices [S1, unofficial].
    *   `_TriangleCount`: Integer count of triangles [S1, unofficial].

**Parameters:**
*   `_GridWidth`, `_GridHeight`, `_GridDepth`: Dimensions of the voxel grid [S1, unofficial].
*   `_CellHalfSize`: Half the side length of a voxel [S1, unofficial].
*   `_BoundsMin`: Minimum corner of the voxelization bounds [S1, unofficial].
*   `_WorldToLocalMatrix`: Transformation matrix to convert world-space ray origins to local space for intersection tests [S1, unofficial].

**Algorithm Steps:**
1.  **Dispatch Check:** For each thread in the compute shader, check if the thread ID exceeds grid dimensions. If so, return [S1, unofficial].
2.  **Grid Iteration:** Iterate through all voxels $(x, y, z)$ in row-major order [S2].
3.  **Skip Empty/Solid:**
    *   If the voxel at $(x, y, z)$ is `Empty`, continue to the next $z$ [S2].
    *   If the voxel is already `Solid` (`isSolid > 0.0`), return immediately [S1, unofficial].
4.  **Find Front Face:** Starting from current $z$, increment `ifront` while `ifront < depth` and `volume[x, y, ifront]` is a `FrontFace` [S2].
5.  **Check Bounds:** If `ifront >= depth`, break the $z$-loop [S2].
6.  **Find Back Face:**
    *   Set `iback = ifront` [S2].
    *   Increment `iback` while `iback < depth` and `volume[x, y, iback]` is `Empty` [S2].
    *   If `iback >= depth`, break the $z$-loop [S2].
    *   If `volume[x, y, iback]` is a `BackFace`, continue incrementing `iback` while it remains a `BackFace` [S2].
7.  **Ray Casting:**
    *   Calculate `cellSize = _CellHalfSize * 2.0` [S1, unofficial].
    *   Compute `rayOrigin` in world space: `id * cellSize + _CellHalfSize + _BoundsMin` [S1, unofficial].
    *   Transform `rayOrigin` to local space using `_WorldToLocalMatrix` [S1, unofficial].
    *   Set `rayDirection = float3(0, 0, 1)` (+Z direction) [S1, unofficial].
    *   Initialize `minDistance = 10000000.0`, `minDistanceNormal = float3(0,0,0)`, `didIntersect = false` [S1, unofficial].
    *   Iterate through all triangles in the mesh:
        *   Load vertices from `_MeshVertices` using `_MeshTriangleIndices` [S1, unofficial].
        *   Test intersection using `IntersectsRayTriangle(rayOrigin, rayDirection, triangle, hitDistance, hitNormal)` [S1, unofficial].
        *   If intersected and `hitDistance < minDistance`, update `minDistance`, `minDistanceNormal`, and set `didIntersect = true` [S1, unofficial].
8.  **Mark Solid:** If `didIntersect` is true and `dot(minDistanceNormal, rayDirection) > 0` (outward-facing), set `_Voxels[voxelIndex].isSolid = 1.0` [S1, unofficial].
9.  **Fill Interval:** Fill all voxels from $z2 = ifront$ to $iback - 1$ with `fill = 1` [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel: FillVolume
# Inputs: _Voxels (RWStructuredBuffer<Voxel>), _MeshVertices (StructuredBuffer<float3>),
#         _MeshTriangleIndices (StructuredBuffer<int>), _TriangleCount (int),
#         _CellHalfSize (float), _GridWidth (int), _GridHeight (int), _GridDepth (int),
#         _BoundsMin (float4), _WorldToLocalMatrix (float4x4)

struct Voxel {
    float isSolid;
    float isFrontFace;
    float isBackFace;
    float fill;
    float3 position;
};

struct Triangle {
    float3 a;
    float3 b;
    float3 c;
};

// Helper: Ray-Triangle Intersection (Möller-Trumbore or similar)
# [G] Standard ray-triangle intersection algorithm. Returns true if hit, sets hitDistance and hitNormal.
def IntersectsRayTriangle(rayOrigin, rayDirection, tri, out hitDistance, out hitNormal):
    # Implementation details omitted for brevity; assumes standard graphics library function
    pass

kernel void VoxelizeMesh(uint3 id : SV_DispatchThreadID) {
    if (id.x >= _GridWidth || id.y >= _GridHeight || id.z >= _GridDepth) return;

    int voxelIndex = id.x + _GridWidth * (id.y + _GridHeight * id.z);
    
    # Check if already solid
    if (_Voxels[voxelIndex].isSolid > 0.0) return;

    # Find front face
    int ifront = id.z;
    while (ifront < _GridDepth && _Voxels[voxelIndex + (ifront - id.z) * _GridWidth * _GridHeight].isFrontFace > 0.0) {
        ifront++;
    }

    if (ifront >= _GridDepth) return;

    # Find back face
    int iback = ifront;
    while (iback < _GridDepth && _Voxels[voxelIndex + (iback - id.z) * _GridWidth * _GridHeight].isBackFace == 0.0 && 
           _Voxels[voxelIndex + (iback - id.z) * _GridWidth * _GridHeight].isSolid == 0.0) {
        iback++;
    }

    if (iback >= _GridDepth) return;

    # If we hit a back face, skip all consecutive back faces
    while (iback < _GridDepth && _Voxels[voxelIndex + (iback - id.z) * _GridWidth * _GridHeight].isBackFace > 0.0) {
        iback++;
    }

    # Ray Casting
    float cellSize = _CellHalfSize * 2.0;
    float3 voxelCenterLocal = float3(id.x, id.y, id.z) * cellSize + float3(_CellHalfSize, _CellHalfSize, _CellHalfSize);
    float3 rayOriginWorld = voxelCenterLocal + _BoundsMin.xyz;
    float3 rayOriginLocal = mul(_WorldToLocalMatrix, float4(rayOriginWorld, 1.0)).xyz;
    float3 rayDirection = float3(0, 0, 1);

    float minDistance = 10000000.0;
    float3 minDistanceNormal = float3(0, 0, 0);
    bool didIntersect = false;

    for (int i = 0; i < _TriangleCount * 3; i += 3) {
        Triangle tri;
        tri.a = _MeshVertices[_MeshTriangleIndices[i]];
        tri.b = _MeshVertices[_MeshTriangleIndices[i+1]];
        tri.c = _MeshVertices[_MeshTriangleIndices[i+2]];

        float hitDistance;
        float3 hitNormal;
        if (IntersectsRayTriangle(rayOriginLocal, rayDirection, tri, hitDistance, hitNormal)) {
            if (hitDistance < minDistance) {
                minDistance = hitDistance;
                minDistanceNormal = hitNormal;
                didIntersect = true;
            }
        }
    }

    if (didIntersect && dot(minDistanceNormal, rayDirection) > 0.0) {
        _Voxels[voxelIndex].isSolid = 1.0;
    }

    # Fill interval
    for (int z2 = ifront; z2 < iback; z2++) {
        int idx = id.x + _GridWidth * (id.y + _GridHeight * z2);
        _Voxels[idx].fill = 1.0;
    }
}
```

## Subtopics

### [Slice Filling (Simple)](1691-slice-filling-simple.md)
Slice Filling (Simple) is a voxelization method that fills the interior of a mesh by processing 2D slices of the voxel grid along one of the three principal axes. It solves the problem of determining which voxels inside the mesh bounds are interior by scanning each slice from both ends toward the center, marking voxels as filled until an existing solid voxel is encountered. It differs from [Solid Voxelization](1695-solid-voxelization.md) by relying on linear edge-to-center scanning within slices rather than global connectivity or flood-fill algorithms, and it serves as the base case for [Slice Filling with BFS Propagation](1909-slice-filling-with-bfs-propagation.md), which extends this linear scan with neighbor propagation.

### [Solid Voxelization](1695-solid-voxelization.md)
Solid Voxelization fills the interior of a watertight 3D mesh with voxels, producing a fully occupied volume rather than just a surface shell. It extends surface voxelization by determining which interior voxels are enclosed by the mesh boundary. Unlike [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md), which relies on Z-axis scanning to identify front-to-back intervals, Solid Voxelization typically employs a two-level grid model or a flood-fill approach to robustly handle complex topologies and singular points.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot T)$, where $N$ is the number of voxels and $T$ is the number of triangles. Each empty voxel potentially tests against all triangles [S1, unofficial].
*   **Memory Complexity:** $O(N + V + I)$, where $N$ is voxel count, $V$ is vertex count, and $I$ is index count [S1, unofficial].
*   **Performance:** Performance is heavily dependent on mesh complexity and grid resolution. High triangle counts lead to significant slowdowns due to per-voxel ray-triangle intersection tests [S1, unofficial].
*   **Strengths:** Simple to implement on GPU using compute shaders; deterministic results for convex or simple concave meshes [S1, unofficial][S2].
*   **Limitations:** Fails on non-watertight meshes or meshes with self-intersections; sensitive to mesh orientation; slow for high-poly meshes [S1, unofficial][S2].
*   **When to Use:** Small to medium complexity meshes where speed of implementation is prioritized over performance [S1, unofficial].
*   **When Not to Use:** High-poly meshes, non-watertight meshes, or when robustness to complex topology is required [S1, unofficial][S3, unofficial].

## Background
Originating from real-time GPU-based voxelization techniques, this method is described in tutorials and code samples for Unity compute shaders [S1, unofficial][S2]. It is part of a pipeline that first generates surface voxels and then fills the interior using ray-casting logic [S1, unofficial][S2].

## Sources
[S1] bzgeb/UnityGPUMeshVoxelizerPart5, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart5, fetched 2026-10-26 (unofficial)
[S2] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
