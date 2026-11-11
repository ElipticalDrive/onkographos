# GPU Mesh Voxelization via AABB-Triangle Intersection

Also called: Compute Shader Triangle Voxelization, ComputeVoxelizer, GPU Voxelizer, Triangle-to-Voxel Rasterization, VoxelizeMesh

## Overview
This method voxelizes a polygonal mesh by dispatching one compute shader thread per voxel in a 3D grid, where each thread performs a brute-force Axis-Aligned Bounding Box (AABB) vs. Triangle intersection test against all mesh triangles [S1, S2, S3, unofficial]. It differs from [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md) by avoiding the overhead of geometry shader primitive generation and from [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) by using a single-writer-per-voxel model without atomic blending [S2, unofficial]. It is chosen when simplicity of implementation is prioritized over performance, as it lacks spatial acceleration structures [S2, unofficial].

## How it works
**Inputs:**
- Input mesh: Array of vertex positions (`_MeshVertices`, `float3`) and triangle indices (`_MeshTriangleIndices`, `int`).
- Grid parameters: `_GridWidth`, `_GridHeight`, `_GridDepth` (integer dimensions); `_CellHalfSize` (float, half-voxel extent); `_BoundsMin` (float4, world-space minimum corner of the grid); `_WorldToLocalMatrix` (float4x4, transforms world to local mesh space) [S1, S3, unofficial].

**Outputs:**
- `_Voxels` (RWStructuredBuffer<Voxel>): 16-byte per voxel. Fields: `position` (float3, world-space center), `isSolid` (float, 1.0 if intersected, 0.0 otherwise) [S1, S3, unofficial].
- Alternative output `_VoxelGridPoints` (RWStructuredBuffer<float4>): x,y,z = world position, w = 1.0/0.0 occupancy [S2, unofficial].
- Alternative output `voxelTexture` (image3D): vec4 diffuse color + alpha 1.0 [S4, unofficial].

**Data Structures:**
- `Triangle`: `a`, `b`, `c` (float3). Size: 36 bytes [S3, unofficial].
- `AABB`: `center` (float3), `extents` (float3). Size: 24 bytes [S3, unofficial].
- `Voxel`: `position` (float3), `isSolid` (float). Size: 16 bytes [S3, unofficial].

**Steps:**
1. **Grid Setup:** Calculate grid dimensions from mesh bounds and `_CellHalfSize`:
   $$xGridSize = \lceil bounds.extents.x / _CellHalfSize \rceil$$
   $$yGridSize = \lceil bounds.extents.y / _CellHalfSize \rceil$$
   $$zGridSize = \lceil bounds.extents.z / _CellHalfSize \rceil$$
   [S2, S3, unofficial].
2. **Dispatch:** Launch compute shader with thread group size $(4, 4, 4)$ [S2, S3, unofficial]. Dispatch counts:
   $$numGroupsX = \lceil xGridSize / 4 \rceil, \quad numGroupsY = \lceil yGridSize / 4 \rceil, \quad numGroupsZ = \lceil zGridSize / 4 \rceil$$
   [S2, S3, unofficial].
3. **Thread Execution (per thread ID $id = (id.x, id.y, id.z)$):**
   a. **Boundary Check:** If $id.x \ge \_GridWidth$ or $id.y \ge \_GridHeight$ or $id.z \ge \_GridDepth$, return [S1, S3, unofficial].
   b. **Voxel AABB Construction:**
      $$cellSize = _CellHalfSize \times 2.0$$
      $$centerPos = (id.x \cdot cellSize + _CellHalfSize + _BoundsMin.x, \quad id.y \cdot cellSize + _CellHalfSize + _BoundsMin.y, \quad id.z \cdot cellSize + _CellHalfSize + _BoundsMin.z)$$
      [S2, S3, unofficial].
      Transform center to local mesh space:
      $$aabb.center = _WorldToLocalMatrix \times centerPos$$
      $$aabb.extents = (_CellHalfSize, _CellHalfSize, _CellHalfSize)$$
      [S3, unofficial].
   c. **Triangle Iteration:** Loop through `_MeshTriangleIndices` in steps of 3. For each triangle $i$, load vertices $a, b, c$ from `_MeshVertices` using indices $i, i+1, i+2$ [S1, S2, S3, unofficial].
   d. **Intersection Test:** Call `IntersectsTriangleAabb(tri, aabb)`. If true, set `intersects = true` and break loop [S1, S2, S3, unofficial].
   e. **Result Storage:** Calculate linear index:
      $$voxelIndex = id.x + _GridWidth \cdot (id.y + _GridHeight \cdot id.z)$$
      [S3, unofficial].
      Write to `_Voxels[voxelIndex]`: `position = centerPos`, `isSolid = intersects ? 1.0 : 0.0` [S3, unofficial].

**Intersection Test (SAT):**
The function `IntersectsTriangleAabb` uses the Separating Axis Theorem [S2, S3, unofficial]. It tests 15 axes [S2, unofficial]:
1. 3 cross products of triangle edges with X, Y, Z axes (9 axes) [S2, S3, unofficial].
2. The 3 X, Y, Z axes themselves (3 axes) [S2, S3, unofficial].
3. The triangle's normal (cross product of two edges) (1 axis) [S2, S3, unofficial].

Helper `IntersectsTriangleAabbSat(v0, v1, v2, aabbExtents, axis)`:
$$p_0 = v_0 \cdot axis, \quad p_1 = v_1 \cdot axis, \quad p_2 = v_2 \cdot axis$$
$$r = aabbExtents.x |axis_x| + aabbExtents.y |axis_y| + aabbExtents.z |axis_z|$$
$$maxP = \max(p_0, p_1, p_2), \quad minP = \min(p_0, p_1, p_2)$$
Return true if $\max(-maxP, minP) \le r$ [S1, S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel: VoxelizeMesh
# Inputs: _MeshVertices (float3[]), _MeshTriangleIndices (int[]), _WorldToLocalMatrix (float4x4)
# Parameters: _GridWidth, _GridHeight, _GridDepth (int), _CellHalfSize (float), _BoundsMin (float4)
# Output: _Voxels (RWStructuredBuffer<Voxel>)

struct Voxel {
    float3 position;
    float isSolid;
};

struct Triangle {
    float3 a, b, c;
};

struct AABB {
    float3 center;
    float3 extents;
};

[numthreads(4, 4, 4)]
void VoxelizeMesh(uint3 id : SV_DispatchThreadID) {
    // 1. Boundary Check
    if (id.x >= _GridWidth || id.y >= _GridHeight || id.z >= _GridDepth) return;

    const float cellSize = _CellHalfSize * 2.0;

    // 2. Voxel AABB Construction
    float3 centerPos = float3(
        id.x * cellSize + _CellHalfSize + _BoundsMin.x,
        id.y * cellSize + _CellHalfSize + _BoundsMin.y,
        id.z * cellSize + _CellHalfSize + _BoundsMin.z
    );

    // Transform to local mesh space for intersection test
    float4 centerPosH = float4(centerPos, 1.0);
    float3 aabbCenterLocal = mul(_WorldToLocalMatrix, centerPosH).xyz;
    AABB aabb;
    aabb.center = aabbCenterLocal;
    aabb.extents = float3(_CellHalfSize, _CellHalfSize, _CellHalfSize);

    bool intersects = false;

    // 3. Triangle Iteration
    for (int i = 0; i < _TriangleCount; i += 3) {
        int idx0 = _MeshTriangleIndices[i];
        int idx1 = _MeshTriangleIndices[i + 1];
        int idx2 = _MeshTriangleIndices[i + 2];

        Triangle tri;
        tri.a = _MeshVertices[idx0];
        tri.b = _MeshVertices[idx1];
        tri.c = _MeshVertices[idx2];

        // 4. Intersection Test
        if (IntersectsTriangleAabb(tri, aabb)) {
            intersects = true;
            break;
        }
    }

    // 5. Result Storage
    int voxelIndex = id.x + _GridWidth * (id.y + _GridHeight * id.z);
    Voxel voxel;
    voxel.position = centerPos;
    voxel.isSolid = intersects ? 1.0 : 0.0;
    _Voxels[voxelIndex] = voxel;
}

bool IntersectsTriangleAabbSat(float3 v0, float3 v1, float3 v2, float3 aabbExtents, float3 axis) {
    float p0 = dot(v0, axis);
    float p1 = dot(v1, axis);
    float p2 = dot(v2, axis);

    float r = aabbExtents.x * abs(axis.x) +
              aabbExtents.y * abs(axis.y) +
              aabbExtents.z * abs(axis.z);

    float maxP = max(p0, max(p1, p2));
    float minP = min(p0, min(p1, p2));

    return !(max(-maxP, minP) > r);
}

bool IntersectsTriangleAabb(Triangle tri, AABB aabb) {
    // Translate triangle to AABB local space
    tri.a -= aabb.center;
    tri.b -= aabb.center;
    tri.c -= aabb.center;

    float3 ab = normalize(tri.b - tri.a);
    float3 bc = normalize(tri.c - tri.b);
    float3 ca = normalize(tri.a - tri.c);

    // 9 Axes: Cross products of edges with X, Y, Z
    float3 a00 = float3(0.0, -ab.z, ab.y); // ab x X
    float3 a01 = float3(0.0, -bc.z, bc.y); // bc x X
    float3 a02 = float3(0.0, -ca.z, ca.y); // ca x X

    float3 a10 = float3(ab.z, 0.0, -ab.x); // ab x Y
    float3 a11 = float3(bc.z, 0.0, -bc.x); // bc x Y
    float3 a12 = float3(ca.z, 0.0, -ca.x); // ca x Y

    float3 a20 = float3(-ab.y, ab.x, 0.0); // ab x Z
    float3 a21 = float3(-bc.y, bc.x, 0.0); // bc x Z
    float3 a22 = float3(-ca.y, ca.x, 0.0); // ca x Z

    // 3 Axes: X, Y, Z
    float3 axisX = float3(1, 0, 0);
    float3 axisY = float3(0, 1, 0);
    float3 axisZ = float3(0, 0, 1);

    // 1 Axis: Triangle Normal
    float3 triNormal = cross(ab, bc);

    // Test all 15 axes
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a00)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a01)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a02)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a10)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a11)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a12)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a20)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a21)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, a22)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, axisX)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, axisY)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, axisZ)) return false;
    if (!IntersectsTriangleAabbSat(tri.a, tri.b, tri.c, aabb.extents, triNormal)) return false;

    return true;
}
```

**Parameters:**
- `_halfSize` / `_CellHalfSize`: Typical value 0.05f [S2, S3, unofficial].
- `_GridWidth`, `_GridHeight`, `_GridDepth`: Integer dimensions [S3, unofficial].
- `_TriangleCount`: Total number of triangle indices [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Triangles:** Normalizing edge vectors in SAT produces NaNs if the triangle has zero area, causing incorrect results or GPU errors [S2, unofficial].
- **Race Conditions:** If multiple triangles write to the same voxel, the last thread wins (overwrite) as there is no atomic blending [S4, unofficial].
- **Missing Axes:** The SAT implementation must check all 15 axes; missing any can lead to false positives [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{cells} \times N_{triangles})$ per voxelization pass, where $N_{cells}$ is the number of voxels and $N_{triangles}$ is the number of triangles [S2, unofficial]. This is brute-force and does not use spatial acceleration structures [S2, unofficial].
- **Memory Complexity:** $O(N_{cells})$ for the output voxel buffer [S3, unofficial].
- **Strengths:** Simple implementation; no spatial data structure maintenance required [S2, unofficial].
- **Limitations:** Poor scalability for high-resolution grids or complex meshes due to $O(N^2)$ complexity [S2, unofficial].
- **When to use:** Small meshes, low-resolution voxelization, or prototyping [S2, unofficial].
- **When not to use:** Large scenes, high-resolution requirements, or real-time performance constraints [S2, unofficial].

## Background
This method is part of the UnityGPUMeshVoxelizer series by bzgeb, starting with Part 2 (2021) and evolving through Part 4 and Part 5 [S2, S3, S1, unofficial]. It is used in Unity-based voxelization pipelines for real-time applications [S2, S3, unofficial].

## Sources
[S1] bzgeb/UnityGPUMeshVoxelizerPart5, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart5, fetched 2026-10-26 (unofficial)
[S2] bzgeb/UnityGpuMeshVoxelizerPart2, https://github.com/bzgeb/UnityGpuMeshVoxelizerPart2, fetched 2026-10-26 (unofficial)
[S3] bzgeb/UnityGPUMeshVoxelizerPart4, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart4, fetched 2026-10-26 (unofficial)
[S4] Oshanath/Voxel-Cone-Tracing, https://github.com/Oshanath/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
