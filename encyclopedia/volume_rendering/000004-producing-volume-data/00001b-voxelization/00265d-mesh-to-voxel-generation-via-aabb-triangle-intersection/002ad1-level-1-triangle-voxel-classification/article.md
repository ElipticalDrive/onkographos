# Level 1 Triangle-Voxel Classification

Also called: CUDAClassifyTessellationKernel

## Overview
Level 1 Triangle-Voxel Classification classifies input triangles into a coarse 3D voxel grid by mapping vertices to voxel indices and testing overlap with all candidate voxels the triangle spans [S1, unofficial]. It identifies candidate voxels via vertex mapping rather than testing every voxel against every triangle, enabling parallel processing where each thread handles one triangle [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `trianglesCUDAData` (flat float array, stride 9 per triangle), `objBoxMin` and `objBoxMax` (scene bounding box, float3), `numDiv` (grid resolution, int3), `boxExtents` (voxel half-extent, float3) [S1, unofficial].
*   **Output:** `voxelTriIndexCUDAData` (list of triangle indices per voxel), `voxelTriCountCUDAData` (count of triangles per voxel), `inOutCUDAData` (auxiliary buffer) [S1, unofficial].

**Data Structures:**
*   `trianglesCUDAData`: A flat array where each triangle is stored as 9 consecutive floats: `[v0.x, v0.y, v0.z, v1.x, v1.y, v1.z, v2.x, v2.y, v2.z]` [S1, unofficial].
*   `voxelTriIndexCUDAData`: A buffer storing triangle indices assigned to each voxel [S1, unofficial].
*   `voxelTriCountCUDAData`: An integer array storing the number of triangles assigned to each voxel [S1, unofficial].
*   `triBufferLen`: The maximum number of triangles allocated per voxel in the buffer [S1, unofficial].

**Algorithm Steps:**
1.  **Thread Assignment:** Each GPU thread processes one triangle [S1, unofficial]. The thread index `t` determines the offset in `trianglesCUDAData` as `t * 9` [S1, unofficial].
2.  **Vertex Extraction:** Extract the three vertices $V_0, V_1, V_2$ from the triangle data [S1, unofficial].
3.  **Vertex-to-Voxel Mapping:** For each vertex $V_i = (x_i, y_i, z_i)$, compute the initial voxel indices $(boxNumX_i, boxNumY_i, boxNumZ_i)$ using the formula:
    $$
    boxNumX_i = \left\lfloor \frac{x_i - objBoxMin.x}{objBoxMax.x - objBoxMin.x} \times numDiv.x \right\rfloor
    $$
    Analogous formulas apply for Y and Z axes [S1, unofficial].
4.  **Boundary Clamping:** If a vertex lies exactly on the maximum boundary, clamp its index to prevent out-of-bounds access:
    ```python
    if boxNumX_i == numDiv.x and vertex.x == objBoxMax.x:
        boxNumX_i -= 1
    ```
    Repeat for Y and Z axes [S1, unofficial].
5.  **Bounding Box Calculation:** Determine the range of voxels spanned by the triangle by computing the minimum and maximum indices across all three vertices:
    $$
    minBoxX = \min(boxNumX_0, boxNumX_1, boxNumX_2)
    $$
    $$
    maxBoxX = \max(boxNumX_0, boxNumX_1, boxNumX_2)
    $$
    Repeat for Y and Z axes [S1, unofficial].
6.  **Voxel Iteration and Overlap Test:** Iterate through all voxels $(p, q, r)$ in the range $[minBoxX, maxBoxX] \times [minBoxY, maxBoxY] \times [minBoxZ, maxBoxZ]$ [S1, unofficial].
    *   Compute the center of the current voxel $(p, q, r)$:
        $$
        boxMidPoint.x = (p + 0.5) \times boxExtents.x \times 2 + objBoxMin.x
        $$
        Analogous for Y and Z axes [S1, unofficial].
    *   Perform a triangle-box intersection test `TriBoxOverlap(boxMidPoint, boxExtents, triVerts)` [S1, unofficial].
    *   If an overlap is detected, compute the linear voxel index:
        $$
        index = r \times numDiv.y \times numDiv.x + q \times numDiv.x + p
        $$
    *   Add the triangle index `t` to the voxel's list using `AddPointToVoxel`, respecting `triBufferLen` to prevent overflow [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CUDAClassifyTessellationKernel(trianglesCUDAData, objBoxMin, objBoxMax, numDiv, boxExtents, triBufferLen):
    numVoxels = numDiv[0] * numDiv[1] * numDiv[2]
    voxelTriCount = [0] * numVoxels
    voxelTriIndex = [[] for _ in range(numVoxels)]

    numTriangles = len(trianglesCUDAData) // 9

    for t in range(numTriangles):
        offset = t * 9
        v0 = [trianglesCUDAData[offset + i] for i in range(3)]
        v1 = [trianglesCUDAData[offset + 3 + i] for i in range(3)]
        v2 = [trianglesCUDAData[offset + 6 + i] for i in range(3)]
        vertices = [v0, v1, v2]

        boxNums = []
        for vertex in vertices:
            bx = int((vertex[0] - objBoxMin[0]) / (objBoxMax[0] - objBoxMin[0]) * numDiv[0])
            by = int((vertex[1] - objBoxMin[1]) / (objBoxMax[1] - objBoxMin[1]) * numDiv[1])
            bz = int((vertex[2] - objBoxMin[2]) / (objBoxMax[2] - objBoxMin[2]) * numDiv[2])

            if bx == numDiv[0] and vertex[0] == objBoxMax[0]:
                bx -= 1
            if by == numDiv[1] and vertex[1] == objBoxMax[1]:
                by -= 1
            if bz == numDiv[2] and vertex[2] == objBoxMax[2]:
                bz -= 1
            boxNums.append((bx, by, bz))

        minBoxX = min(b[0] for b in boxNums)
        maxBoxX = max(b[0] for b in boxNums)
        minBoxY = min(b[1] for b in boxNums)
        maxBoxY = max(b[1] for b in boxNums)
        minBoxZ = min(b[2] for b in boxNums)
        maxBoxZ = max(b[2] for b in boxNums)

        for p in range(minBoxX, maxBoxX + 1):
            for q in range(minBoxY, maxBoxY + 1):
                for r in range(minBoxZ, maxBoxZ + 1):
                    midX = (p + 0.5) * boxExtents[0] * 2 + objBoxMin[0]
                    midY = (q + 0.5) * boxExtents[1] * 2 + objBoxMin[1]
                    midZ = (r + 0.5) * boxExtents[2] * 2 + objBoxMin[2]
                    boxMidPoint = [midX, midY, midZ]

                    if TriBoxOverlap(boxMidPoint, boxExtents, vertices):
                        index = r * numDiv[1] * numDiv[0] + q * numDiv[0] + p
                        AddPointToVoxel(index, t, voxelTriIndex, voxelTriCount, triBufferLen)

def TriBoxOverlap(boxMidPoint, boxHalfExtent, triVerts):
    # Standard Separating Axis Theorem triangle-box overlap test [G]
    # ... (simplified SAT implementation)
    pass # Note: Standard SAT implementation omitted for brevity; returns True if triangle intersects box [G]

def AddPointToVoxel(index, triIndex, voxelTriIndex, voxelTriCount, triBufferLen):
    if voxelTriCount[index] < triBufferLen:
        voxelTriIndex[index].append(triIndex)
        voxelTriCount[index] += 1
```

**Parameters:**
*   `triBufferLen`: Maximum number of triangles per voxel [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Precision:** Vertices exactly at `objBoxMax` must be clamped to avoid index out-of-bounds errors [S1, unofficial].
*   **Degenerate Triangles:** The method assumes valid triangles; degenerate cases may produce incorrect overlaps or empty ranges [G].
*   **Buffer Overflow:** If a voxel exceeds `triBufferLen`, additional triangles are dropped or require a dynamic resizing strategy, which is not detailed in the source [S1, unofficial].

## Subtopics

### [Level 2 Triangle-Voxel Classification](5512-level-2-triangle-voxel-classification.md)
Refines the coarse classification by re-evaluating overlaps within sub-voxels to determine precise inside/outside status. It uses Level 1 results to limit the search space, processing only triangles assigned to the parent Level 1 box. Choose this when high-precision topology resolution is required.

### [Occupancy Grid Voxelization](6958-occupancy-grid-voxelization.md)
A brute-force method that checks every voxel against every triangle. It prioritizes implementation simplicity and exact boundary definition over speed. Choose this for small meshes or when hierarchical overhead is unjustified.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method processes one triangle per thread and performs a constant-time overlap test per candidate voxel [S1, unofficial].
*   **Memory Complexity:** Scales with the total number of voxels and `triBufferLen` [S1, unofficial].
*   **Performance:** Highly parallelizable on GPU, with one thread per triangle [S1, unofficial]. Performance scales with the number of triangles and the spatial coherence of the mesh [S1, unofficial].
*   **Strengths:** Efficient reduction of search space for subsequent steps; simple implementation; GPU-friendly [S1, unofficial].
*   **Limitations:** Coarse classification may include false positives (voxels that overlap the triangle's bounding box but not the triangle itself); requires careful handling of boundary conditions [S1, unofficial].
*   **When to Use:** As the first stage in hierarchical voxelization pipelines where fine-grained classification is computationally expensive [S1, unofficial].
*   **When Not to Use:** For very small meshes where brute-force is faster; when exact boundary definition is required without further refinement [S1, unofficial].

## Background
The method is implemented as a CUDA kernel in the provided source code [S1, unofficial]. It is part of a broader effort to accelerate mesh processing for real-time visualization and volume rendering applications [S1, unofficial].

## Sources
[S1] idealab-isu/GPView, https://github.com/idealab-isu/GPView, fetched 2026-10-26 (unofficial)
