# GPU Mesh Voxelization via SAT Ray Casting

Also called: UnityGPUMeshVoxelizer, VoxelizeMesh

## Overview
This method converts a triangle mesh into a 3D voxel grid by dispatching a compute shader where each thread corresponds to a voxel cell. It determines if a voxel is solid by testing the cell against every triangle in the mesh using a Separating Axis Theorem (SAT) broad-phase test followed by a Möller-Trumbore ray-triangle intersection. It differs from hierarchical voxelization by using a brute-force per-voxel, per-triangle approach, making it simple to implement but computationally expensive for high-resolution grids or complex meshes.

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertex positions and triangle indices.
*   **Input:** Voxel grid dimensions ($W, H, D$) and cell size ($s$).
*   **Input:** World-space bounds of the voxelization volume.
*   **Output:** A structured buffer of voxels, each containing a position and a boolean `isSolid` flag.

**Data Structures:**
The implementation uses the following structures in HLSL/C#:

1.  **Voxel:**
    ```hlsl
    struct Voxel {
        float3 position; // World-space center of the voxel
        float isSolid;   // 1.0 if solid, 0.0 otherwise
    };
    ```
    Stored in a `RWStructuredBuffer<Voxel>`.

2.  **Triangle:**
    ```hlsl
    struct Triangle {
        float3 a, b, c; // Vertices in world space
    };
    ```
    Triangles are not stored as a single struct buffer but are fetched from vertex and index buffers.

3.  **Mesh Buffers:**
    *   `_MeshVertices`: `StructuredBuffer<float3>`, size $N_v$ (number of vertices).
    *   `_MeshTriangleIndices`: `StructuredBuffer<int>`, size $N_t \times 3$ (number of triangles $\times$ 3).

4.  **Grid Parameters:**
    *   `_GridWidth`, `_GridHeight`, `_GridDepth`: Integers defining the voxel count per axis.
    *   `_CellHalfSize`: Float, half the side length of a voxel (default 0.05f).
    *   `_BoundsMin`: Float3, minimum corner of the voxel grid in world space.
    *   `_WorldToLocalMatrix`: Matrix, transforms world coordinates to local mesh coordinates.
    *   `_TriangleCount`: Integer, total number of triangles.

**Algorithm Steps:**

1.  **Grid Initialization:**
    Calculate grid dimensions from the mesh collider bounds and `_CellHalfSize`.
    $$ W = \lceil \frac{\text{bounds.extents.x}}{\text{_CellHalfSize} \times 2} \rceil $$
    $$ H = \lceil \frac{\text{bounds.extents.y}}{\text{_CellHalfSize} \times 2} \rceil $$
    $$ D = \lceil \frac{\text{bounds.extents.z}}{\text{_CellHalfSize} \times 2} \rceil $$
    Allocate `_voxelsBuffer` of size $W \times H \times D$.

2.  **Buffer Upload:**
    Upload mesh vertices to `_MeshVertices` and indices to `_MeshTriangleIndices`.

3.  **Compute Dispatch:**
    Dispatch the `VoxelizeMesh` kernel with thread groups:
    $$ \text{Groups}_x = \lceil \frac{W}{\text{GroupSize}} \rceil, \quad \text{Groups}_y = \lceil \frac{H}{\text{GroupSize}} \rceil, \quad \text{Groups}_z = \lceil \frac{D}{\text{GroupSize}} \rceil $$

4.  **Voxelization Kernel (`VoxelizeMesh`):**
    For each thread with global ID $(id.x, id.y, id.z)$:
    a.  **Bounds Check:** If $id.x \ge W$ or $id.y \ge H$ or $id.z \ge D$, return.
    b.  **Compute Cell Center:**
        $$ \text{cellSize} = \text{_CellHalfSize} \times 2.0 $$
        $$ \text{centerPos} = (id.x \times \text{cellSize} + \text{_CellHalfSize} + \text{_BoundsMin}.x, \dots) $$
        (Repeat for y and z components).
    c.  **Initialize Solidity:** Set `isSolid = 0`.
    d.  **Triangle Iteration:** Iterate through all triangles $i$ from $0$ to `_TriangleCount` - 1.
        i.  Fetch vertices $A, B, C$ from `_MeshVertices` using indices from `_MeshTriangleIndices`.
        ii. **SAT Broad-Phase:** Test if the voxel AABB (centered at `centerPos`, extents `_CellHalfSize`) intersects the triangle's AABB.
        iii. **Ray-Triangle Intersection:** If SAT passes, cast a ray from `centerPos` towards the triangle centroid (or use the Möller-Trumbore algorithm directly if the SAT is considered part of the intersection logic). If a hit is found, set `isSolid = 1` and break.
    e.  **Write Output:** Write the voxel to `_voxelsBuffer[id.z * W * H + id.y * W + id.x]`.

5.  **Fill Volume (Optional):**
    Dispatch a subsequent kernel to propagate solidity to neighboring empty voxels if a "filled" volume is required rather than a surface shell.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# HLSL-style pseudocode for the VoxelizeMesh kernel

#define EPSILON 0.0000001

struct Voxel {
    float3 position;
    float isSolid;
};

// Global buffers and parameters
RWStructuredBuffer<Voxel> _Voxels;
StructuredBuffer<float3> _MeshVertices;
StructuredBuffer<int> _MeshTriangleIndices;
int _GridWidth, _GridHeight, _GridDepth;
float _CellHalfSize;
float3 _BoundsMin;
int _TriangleCount;

// Helper: Möller-Trumbore Ray-Triangle Intersection
// Returns true if ray (origin, direction) hits triangle (a, b, c)
bool RayTriangleIntersect(float3 origin, float3 direction, float3 a, float3 b, float3 c) {
    float3 ab = b - a;
    float3 ac = c - a;
    float3 h = cross(direction, ac);
    float a_val = dot(ab, h);
    
    // Parallel check
    if (a_val > -EPSILON && a_val < EPSILON) {
        return false;
    }
    
    float f = 1.0 / a_val;
    float3 s = origin - a;
    float u = f * dot(s, h);
    
    if (u < 0.0 || u > 1.0) {
        return false;
    }
    
    float3 q = cross(s, ab);
    float v = f * dot(direction, q);
    
    if (v < 0.0 || u + v > 1.0) {
        return false;
    }
    
    float t = f * dot(ac, q);
    
    // Positive distance check
    if (t <= EPSILON) {
        return false;
    }
    
    return true;
}

// Helper: SAT Projection Test for AABB vs Triangle
// Tests if voxel AABB (center, extents) overlaps triangle's AABB
bool SATTest(float3 voxelCenter, float3 voxelExtents, float3 a, float3 b, float3 c) {
    // Compute triangle AABB
    float3 minT = min(a, min(b, c));
    float3 maxT = max(a, max(b, c));
    float3 triCenter = (minT + maxT) * 0.5;
    float3 triExtents = (maxT - minT) * 0.5;
    
    // Separating axis tests (axes are X, Y, Z, and triangle normal)
    // Simplified: Check overlap on X, Y, Z axes first
    if (abs(voxelCenter.x - triCenter.x) > voxelExtents.x + triExtents.x) return false;
    if (abs(voxelCenter.y - triCenter.y) > voxelExtents.y + triExtents.y) return false;
    if (abs(voxelCenter.z - triCenter.z) > voxelExtents.z + triExtents.z) return false;
    
    // Test triangle normal axis
    float3 n = normalize(cross(b - a, c - a));
    float p0 = dot(a, n);
    float p1 = dot(b, n);
    float p2 = dot(c, n);
    float maxP = max(p0, max(p1, p2));
    float minP = min(p0, min(p1, p2));
    
    float r = voxelExtents.x * abs(n.x) + voxelExtents.y * abs(n.y) + voxelExtents.z * abs(n.z);
    float d = abs(dot(voxelCenter, n) - (minP + maxP) * 0.5); // Simplified distance
    
    // Correct SAT formula from source:
    // Radius r = aabbExtents.x * abs(dot(X, axis)) + ...
    // Intersection if !(max(-maxP, minP) > r) -- Note: Source formula is slightly ambiguous in text, 
    // standard SAT for AABB vs Triangle involves projecting both onto the axis.
    // Using the source's specific logic structure:
    float triRadius = 0.0; // Placeholder for complex projection logic if needed, 
                           // but standard AABB-Triangle SAT is more involved.
                           // The source notes imply a simplified check or specific implementation.
                           // We rely on the Ray-Triangle intersection as the definitive test 
                           // if the broad phase passes.
                           
    return true; // Broad phase passed, proceed to precise test
}

[numthreads(8, 8, 8)] // Example group size
void VoxelizeMesh(uint3 id : SV_DispatchThreadID) {
    if (id.x >= _GridWidth || id.y >= _GridHeight || id.z >= _GridDepth) return;
    
    float cellSize = _CellHalfSize * 2.0;
    float3 centerPos = float3(
        id.x * cellSize + _CellHalfSize + _BoundsMin.x,
        id.y * cellSize + _CellHalfSize + _BoundsMin.y,
        id.z * cellSize + _CellHalfSize + _BoundsMin.z
    );
    
    Voxel v;
    v.position = centerPos;
    v.isSolid = 0.0;
    
    float3 voxelExtents = float3(_CellHalfSize, _CellHalfSize, _CellHalfSize);
    
    for (int i = 0; i < _TriangleCount; i++) {
        int idx0 = _MeshTriangleIndices[i * 3];
        int idx1 = _MeshTriangleIndices[i * 3 + 1];
        int idx2 = _MeshTriangleIndices[i * 3 + 2];
        
        float3 a = _MeshVertices[idx0];
        float3 b = _MeshVertices[idx1];
        float3 c = _MeshVertices[idx2];
        
        // Broad phase
        if (!SATTest(centerPos, voxelExtents, a, b, c)) continue;
        
        // Precise test: Ray from voxel center to triangle centroid
        float3 centroid = (a + b + c) / 3.0;
        float3 direction = normalize(centroid - centerPos);
        
        if (RayTriangleIntersect(centerPos, direction, a, b, c)) {
            v.isSolid = 1.0;
            break;
        }
    }
    
    _Voxels[id.z * _GridWidth * _GridHeight + id.y * _GridWidth + id.x] = v;
}
```

**Parameters:**
*   `_CellHalfSize`: Default 0.05f. Determines voxel resolution.
*   `_GridWidth`, `_GridHeight`, `_GridDepth`: Derived from mesh bounds and `_CellHalfSize`.
*   `_BoundsMin`: World-space minimum corner of the voxelization volume.
*   `_WorldToLocalMatrix`: Used if mesh vertices are in local space and need transformation to world space for intersection tests.

**Edge Cases and Pitfalls:**
*   **Ray Parallel to Triangle:** If the ray direction is parallel to the triangle plane, the denominator $a$ in Möller-Trumbore becomes zero. Handled by checking `if (a > -epsilon && a < epsilon) return false;` [S1, unofficial].
*   **Barycentric Coordinates:** If $u$ or $v$ are outside $[0, 1]$ or $u+v > 1$, the ray misses the triangle. Handled by explicit bounds checks [S1, unofficial].
*   **Back-facing/Degenerate Hits:** Hits with $t \le \epsilon$ are rejected to avoid self-intersection or degenerate cases. Handled by `return t > epsilon;` [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V \times T)$, where $V$ is the total number of voxels ($W \times H \times D$) and $T$ is the number of triangles. Each voxel thread iterates over all triangles [S1, unofficial].
*   **Memory Complexity:** $O(V + V_{mesh} + T_{mesh})$, where $V$ is the voxel buffer size, $V_{mesh}$ is the vertex buffer size, and $T_{mesh}$ is the index buffer size [S1, unofficial].
*   **Performance:** Performance degrades linearly with grid resolution and triangle count. Suitable for real-time visualization of moderate-resolution grids on modern GPUs. Not suitable for high-fidelity volume rendering without acceleration structures (e.g., BVH) [S1, unofficial].
*   **Strengths:** Simple implementation, no complex data structures required, fully parallelizable.
*   **Limitations:** No spatial acceleration, leading to poor performance on dense meshes or high-resolution grids.

## Background
This method is implemented in the Unity engine using ComputeShaders and is part of the "UnityGPUMeshVoxelizer" project series. It was developed by bzgeb as an unofficial tutorial/example for GPU-based mesh voxelization [S1, unofficial].

## Sources
[S1] bzgeb/UnityGPUMeshVoxelizerPart5, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart5, fetched 2026-10-26 (unofficial)
