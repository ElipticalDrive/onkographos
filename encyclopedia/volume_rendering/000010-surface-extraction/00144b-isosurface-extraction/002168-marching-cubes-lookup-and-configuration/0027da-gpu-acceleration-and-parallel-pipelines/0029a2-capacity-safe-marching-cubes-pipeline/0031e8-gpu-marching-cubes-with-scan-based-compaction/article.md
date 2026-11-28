# GPU Marching Cubes with Scan-Based Compaction

Also called: Empty Space Skipping via Stream Compaction, MarchingCubesGPU, Stream Compaction for Empty Space Skipping

## Overview
A GPU-accelerated Marching Cubes pipeline that classifies voxels, uses exclusive prefix sums to count active voxels and required vertices, compacts the voxel index array to remove empty voxels, and extracts the isosurface only for active voxels [S1, unofficial]. It optimizes Marching Cubes by using stream compaction to avoid processing voxels that do not contain the isosurface [S2]. Builds on [Capacity-Safe Marching Cubes Pipeline](4596-capacity-safe-marching-cubes-pipeline.md) [S1, unofficial].

## How it works
**Inputs:**
- `basePoint`: World-space origin of the voxel grid (`float3`) [S1, unofficial].
- `gridSize`: Dimensions of the voxel grid in voxels (`uint3`) [S1, unofficial].
- `voxelSize`: Physical size of each voxel (`float3`) [S1, unofficial].
- `isoValue`: Scalar threshold for isosurface extraction (`float`, default `0.2f`) [S1, unofficial].
- Implicit input: A scalar field function $V(x, y, z)$ evaluated per voxel corner [S1, unofficial].

**Outputs:**
- `resultPts`: Array of `float3` vertices representing the extracted isosurface [S1, unofficial].
- `successful`: Boolean indicating if extraction succeeded (false if no active voxels) [S1, unofficial].

**Data Structures:**
Device arrays (all size `numVoxels` unless noted):
- `d_voxelVerts`: `uint*`, stores the number of vertices generated per voxel [S1, unofficial].
- `d_voxelOccupied`: `uint*`, binary occupancy flag per voxel (1 if active, 0 if empty) [S1, unofficial].
- `d_voxelVertsScan`: `uint*`, exclusive prefix sum of `d_voxelVerts` [S1, unofficial].
- `d_voxelOccupiedScan`: `uint*`, exclusive prefix sum of `d_voxelOccupied` [S1, unofficial].
- `d_compVoxelArray`: `uint*`, compacted array of indices for active voxels [S1, unofficial].
- `d_result`: `float3*`, output vertex buffer, size `num_resultVertices * sizeof(float3)` [S1, unofficial].

Host pinned arrays:
- `d_voxelOccupied`: `uint*`, size `numVoxels` [S1, unofficial].
- `d_voxelVerts`: `uint*`, size `numVoxels` [S1, unofficial].

**Steps:**

1. **Allocation**: Allocate device arrays `d_voxelVerts`, `d_voxelOccupied`, `d_voxelVertsScan`, `d_voxelOccupiedScan`, `d_compVoxelArray` and pinned host arrays `d_voxelOccupied`, `d_voxelVerts` of size `sizeof(uint) * numVoxels` [S1, unofficial].

2. **Voxel Classification**: Launch `classifyVoxel` kernel [S1, unofficial].
   - Grid configuration: `dim3 grid((numVoxels + threads - 1) / threads, 1, 1)` [S1, unofficial]. If `grid.x > 65535`, split into 2D: `grid.y = grid.x / 32768; grid.x = 32768;` [S1, unofficial].
   - Block size: `threads = 256` [S1, unofficial].
   - Each thread processes one voxel index $i$ [S1, unofficial].
   - Compute world coordinates of the 8 corners of voxel $i$ [S1, unofficial].
   - Evaluate the scalar field at each corner [S1, unofficial].
   - Determine if the isosurface intersects the voxel (standard Marching Cubes classification) [S1, unofficial].
   - Write `1` to `d_voxelOccupied[i]` if active, `0` otherwise [S1, unofficial].
   - Write the number of vertices generated for this voxel to `d_voxelVerts[i]` [S1, unofficial].

3. **Occupancy Scan**: Run `exclusiveSumScan` on `d_voxelOccupied` to produce `d_voxelOccupiedScan` [S1, unofficial].

4. **Active Voxel Count**:
   - Copy the last element of `d_voxelOccupied` to host variable `lastElement` [S1, unofficial].
   - Copy the last element of `d_voxelOccupiedScan` to host variable `lastScanElement` [S1, unofficial].
   - Compute `num_activeVoxels = lastElement + lastScanElement` [S1, unofficial].
   - If `num_activeVoxels == 0`, set `num_resultVertices = 0`, mark `successful = false`, and return [S1, unofficial].

5. **Voxel Compaction**: Launch `compactVoxels` kernel [S1, unofficial].
   - Grid: Same as classification [S1, unofficial].
   - Each thread $i$ checks `d_voxelOccupied[i]`. If true, write $i$ to `d_compVoxelArray` at an index determined by the prefix sum. Specifically, the output index is `d_voxelOccupiedScan[i]` [S1, unofficial].

6. **Vertex Count Scan**: Run `exclusiveSumScan` on `d_voxelVerts` to produce `d_voxelVertsScan` [S1, unofficial].

7. **Result Vertex Count**:
   - Copy the last element of `d_voxelVerts` to host variable `lastElement2` [S1, unofficial].
   - Copy the last element of `d_voxelVertsScan` to host variable `lastScanElement2` [S1, unofficial].
   - Compute `num_resultVertices = lastElement2 + lastScanElement2` [S1, unofficial].

8. **Result Buffer Allocation**: Allocate `d_result` of size `num_resultVertices * sizeof(float3)` [S1, unofficial].

9. **Isosurface Extraction**: Launch `extractIsosurface` kernel [S1, unofficial].
   - Grid: `dim3 grid2((num_activeVoxels + threads - 1) / threads, 1, 1)`. If `grid2.x > 65535`, split: `grid2.y = grid2.x / 32768; grid2.x = 32768;` [S1, unofficial]. Note: The source mentions a potential bug where extraction code checks `grid.x` instead of `grid2.x` in the condition, but the correct logic is to check the extraction grid [S1, unofficial].
   - Each thread $j$ corresponds to an active voxel index $idx = d\_compVoxelArray[j]$ [S1, unofficial].
   - Retrieve the 8 corner coordinates of voxel $idx$ [S1, unofficial].
   - Evaluate the scalar field at corners [S1, unofficial].
   - Determine the Marching Cubes case index [S1, unofficial].
   - Look up the triangle configuration from the MC lookup table [S1, unofficial].
   - Compute intersection points on edges using linear interpolation [S1, unofficial].
   - Write the resulting vertices to `d_result` at offsets provided by `d_voxelVertsScan[idx]` [S1, unofficial].

10. **Copy Result**: Copy `d_result` to host `resultPts` [S1, unofficial].

11. **Cleanup**: Free pinned memory (`cudaFreeHost`) and device arrays (`cudaFree`) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_marching_cubes_scan_compact(basePoint, gridSize, voxelSize, isoValue):
    threads = 256
    numVoxels = gridSize.x * gridSize.y * gridSize.z
    
    # 1. Allocate device arrays
    d_voxelVerts = [0] * numVoxels
    d_voxelOccupied = [0] * numVoxels
    d_voxelVertsScan = [0] * numVoxels
    d_voxelOccupiedScan = [0] * numVoxels
    d_compVoxelArray = [0] * numVoxels
    
    # 2. Classify Voxels
    grid_x = (numVoxels + threads - 1) // threads
    grid_y = 1
    if grid_x > 65535:
        grid_y = grid_x // 32768
        grid_x = 32768
    grid = (grid_x, grid_y, 1)
    
    classifyVoxel(d_voxelVerts, d_voxelOccupied, gridSize, numVoxels, basePoint, voxelSize, isoValue)
    
    # 3. Scan Occupancy
    exclusiveSumScan(d_voxelOccupiedScan, d_voxelOccupied, numVoxels)
    
    # 4. Count Active Voxels
    lastElement = d_voxelOccupied[-1]
    lastScanElement = d_voxelOccupiedScan[-1]
    num_activeVoxels = lastElement + lastScanElement
    
    if num_activeVoxels == 0:
        num_resultVertices = 0
        successful = False
        return successful, []

    # 5. Compact Voxels
    compactVoxels(d_compVoxelArray, d_voxelOccupied, d_voxelOccupiedScan, numVoxels)
    
    # 6. Scan Vertex Counts
    exclusiveSumScan(d_voxelVertsScan, d_voxelVerts, numVoxels)
    
    # 7. Count Result Vertices
    lastElement2 = d_voxelVerts[-1]
    lastScanElement2 = d_voxelVertsScan[-1]
    num_resultVertices = lastElement2 + lastScanElement2
    
    # 8. Allocate Result Buffer
    d_result = [None] * num_resultVertices
    
    # 9. Extract Isosurface
    grid2_x = (num_activeVoxels + threads - 1) // threads
    grid2_y = 1
    if grid2_x > 65535:
        grid2_y = grid2_x // 32768
        grid2_x = 32768
    grid2 = (grid2_x, grid2_y, 1)
    
    extractIsosurface(d_result, d_compVoxelArray, d_voxelVertsScan, num_activeVoxels, gridSize, basePoint, voxelSize, isoValue)
    
    # 10. Copy Result
    resultPts = list(d_result)
    
    # 11. Cleanup
    successful = True
    return successful, resultPts

def classifyVoxel(d_voxelVerts, d_voxelOccupied, gridSize, numVoxels, basePoint, voxelSize, isoValue):
    for i in range(numVoxels):
        x = i % gridSize.x
        y = (i // gridSize.x) % gridSize.y
        z = i // (gridSize.x * gridSize.y)
        
        corners = []
        for dx in [0, 1]:
            for dy in [0, 1]:
                for dz in [0, 1]:
                    wx = basePoint.x + (x + dx) * voxelSize.x
                    wy = basePoint.y + (y + dy) * voxelSize.y
                    wz = basePoint.z + (z + dz) * voxelSize.z
                    corners.append((wx, wy, wz))
        
        values = []
        for (wx, wy, wz) in corners:
            val = cos(wx) * sin(wy) + cos(wy) * sin(wz) + cos(wz) * sin(wx)
            values.append(val)
        
        min_val = min(values)
        max_val = max(values)
        
        if min_val <= isoValue and max_val >= isoValue:
            d_voxelOccupied[i] = 1
            case_index = 0
            for k, v in enumerate(values):
                if v > isoValue:
                    case_index |= (1 << k)
            num_triangles = mc_table_num_triangles[case_index]
            d_voxelVerts[i] = num_triangles * 3
        else:
            d_voxelOccupied[i] = 0
            d_voxelVerts[i] = 0

def exclusiveSumScan(output, input, size):
    acc = 0
    for i in range(size):
        output[i] = acc
        acc += input[i]

def compactVoxels(d_compVoxelArray, d_voxelOccupied, d_voxelOccupiedScan, numVoxels):
    for i in range(numVoxels):
        if d_voxelOccupied[i] == 1:
            out_idx = d_voxelOccupiedScan[i]
            d_compVoxelArray[out_idx] = i

def extractIsosurface(d_result, d_compVoxelArray, d_voxelVertsScan, num_activeVoxels, gridSize, basePoint, voxelSize, isoValue):
    for j in range(num_activeVoxels):
        idx = d_compVoxelArray[j]
        
        x = idx % gridSize.x
        y = (idx // gridSize.x) % gridSize.y
        z = idx // (gridSize.x * gridSize.y)
        
        corners = []
        for dx in [0, 1]:
            for dy in [0, 1]:
                for dz in [0, 1]:
                    wx = basePoint.x + (x + dx) * voxelSize.x
                    wy = basePoint.y + (y + dy) * voxelSize.y
                    wz = basePoint.z + (z + dz) * voxelSize.z
                    corners.append((wx, wy, wz))
        
        values = []
        for (wx, wy, wz) in corners:
            val = cos(wx) * sin(wy) + cos(wy) * sin(wz) + cos(wz) * sin(wx)
            values.append(val)
        
        case_index = 0
        for k, v in enumerate(values):
            if v > isoValue:
                case_index |= (1 << k)
        
        triangles = mc_table_triangles[case_index]
        offset = d_voxelVertsScan[idx]
        point_idx = 0
        for tri in triangles:
            for edge_idx in tri:
                p = interpolate_edge(edge_idx, corners, values, isoValue)
                d_result[offset + point_idx] = p
                point_idx += 1

def interpolate_edge(edge_idx, corners, values, isoValue):
    edge_pairs = [
        (0,1), (1,2), (2,3), (3,0),
        (4,5), (5,6), (6,7), (7,4),
        (0,4), (1,5), (2,6), (3,7)
    ]
    if edge_idx < 0 or edge_idx >= 12:
        return (0,0,0)
    
    c1_idx, c2_idx = edge_pairs[edge_idx]
    c1 = corners[c1_idx]
    c2 = corners[c2_idx]
    v1 = values[c1_idx]
    v2 = values[c2_idx]
    
    denom = v2 - v1
    if denom == 0:
        return c1
    
    t = (isoValue - v1) / denom
    return (
        c1[0] + t * (c2[0] - c1[0]),
        c1[1] + t * (c2[1] - c1[1]),
        c1[2] + t * (c2[2] - c1[2])
    )
```

## Subtopics

### [QEF-based Octree Mesh Generation](16281-qef-based-octree-mesh-generation.md)
A GPU compute pipeline that converts an octree of leaf nodes (extracted from a density field) into a compact triangle mesh using parallel index generation, prefix-sum compaction, and vertex attribute assembly. It builds on [GPU Marching Cubes with Scan-Based Compaction](6632-gpu-marching-cubes-with-scan-based-compaction.md) by handling the final assembly of vertex attributes and index compaction for the extracted geometry.

## Operating characteristics
- **Time Complexity**: $O(N)$ for classification, scan, and extraction, where $N$ is the number of voxels. The scans are parallel prefix sums, typically $O(\log N)$ depth with $O(N)$ work.
- **Memory Complexity**: $O(N)$ for device arrays (`d_voxelVerts`, `d_voxelOccupied`, etc.) and $O(V)$ for the result buffer, where $V$ is the number of output vertices.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales with the number of voxels and GPU cores. The 2D grid splitting handles large voxel counts exceeding 65535 in one dimension [S1, unofficial].
- **Strengths**: Avoids buffer overflow by pre-calculating exact vertex counts. Skips empty voxels, reducing computation for sparse fields.
- **Limitations**: Requires multiple kernel launches and memory transfers. The naive vertex counting may overestimate if shared vertices are not handled locally. The reference implementation uses a specific scalar field function, not a general texture lookup [S1, unofficial].
- **When to use**: When the voxel grid is large and sparse, or when exact buffer sizing is required to avoid reallocation.
- **When not to use**: When the field is dense (many active voxels), the overhead of compaction may not be worth it. When vertex sharing/deduplication is critical for mesh quality, this method produces a non-manifold or duplicated vertex mesh unless post-processed.

## Background
It is described as an optimization in Marching Cubes using stream compaction to avoid processing empty voxels [S2]. It is used in volume rendering and medical imaging contexts [S2].

## Sources
[S1] AlbertLiDesign/ALG_MarchingCubes_GPU, https://github.com/AlbertLiDesign/ALG_MarchingCubes_GPU, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering and Medical Imaging - DeepWiki, https://deepwiki.com/NVIDIA/cuda-samples/11.3-volume-rendering-and-medical-imaging, fetched 2026-10-29
