# Chunked Marching Cubes Terrain Generation

Also called: Destructible infinite marching cubes terrain, Eldemarkki Marching Cubes Terrain, Infinite Marching Cubes, Marching Job

## Overview
A chunked implementation of the Marching Cubes algorithm designed for real-time, infinite, and destructible voxel terrain generation [S1, unofficial]. It processes voxels in parallel chunks to enable dynamic terrain modification and infinite world streaming [S1, unofficial]. It differs from standard Marching Cubes by dividing the volume into independent, reusable chunks and leveraging a job-based parallel execution model to handle density generation and mesh extraction concurrently [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D density field (noise map) stored as a 1D array of floats, representing voxel densities. The field dimensions are $(size+1)^3$ for a chunk of grid dimension $size$.
*   **Output:** A `Mesh` object containing vertices and triangle indices, plus a baked collider.

**Data Structures:**
1.  **Density Map (`noiseMap`):** A `NativeArray<float>` of size $(size+1)^3$. Indexed linearly as:
    $$ \text{densityIndex} = x \cdot (size+1)^2 + y \cdot (size+1) + z $$
    where $(x,y,z)$ are local voxel coordinates within the chunk.
2.  **MarchingJob Struct:**
    *   `densities`: `NativeArray<float>` (read-only).
    *   `isolevel`: `float` (read-only, typically 0.0).
    *   `chunkSize`: `int` (read-only).
    *   `counter`: `Counter` (write-only, atomic integer for vertex indexing).
    *   `vertices`: `NativeArray<Vector3>` (write-only, parallel restriction disabled).
    *   `triangles`: `NativeArray<int>` (write-only, parallel restriction disabled).
3.  **Counter:** An atomic integer structure using `Interlocked.Increment` to safely track the global vertex index across parallel jobs [S1, unofficial].
4.  **Lookup Tables:** Inherits `edgeTable` and `triangleTable` from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md).

**Algorithm Steps:**

1.  **Initialize Chunk:**
    *   Allocate `buffer` (vertices), `indexes` (triangles), and `noiseMap` (densities).
    *   Allocate `Counter` with `Allocator.Persistent`.
    *   Create `Mesh` and `MeshFilter`. Set `SubMeshDescriptor` topology to `MeshTopology.Triangles`.

2.  **Generate Density Field (NoiseJob):**
    *   Schedule a job to populate `noiseMap`.
    *   Work size: $(size+1)^3$.
    *   Chunk size: 64.
    *   Parameters: amplitude, frequency, octaves, offset, seed, surface level.

3.  **Execute Marching Cubes (MarchingJob):**
    *   Schedule `MarchingJob` dependent on the completion of the NoiseJob.
    *   Work size: $size^3$.
    *   Chunk size: 32.
    *   For each voxel index in the job:
        1.  **Calculate Local Position:**
            $$ \text{localPos} = (\text{index} / (size \cdot size), \ (\text{index} / size) \% size, \ \text{index} \% size) $$
        2.  **Get Corner Densities:**
            For each of the 8 corners $i \in [0,7]$:
            $$ \text{voxelCorner} = \text{localPos} + \text{CubeCorners}[i] $$
            $$ \text{densities}[i] = \text{noiseMap}[\text{voxelCorner}.x \cdot (size+1)^2 + \text{voxelCorner}.y \cdot (size+1) + \text{voxelCorner}.z] $$
        3.  **Calculate Case Index:**
            $$ \text{cubeIndex} = 0 $$
            For $i$ from 0 to 7:
            If $\text{densities}[i] < \text{isolevel}$:
            $$ \text{cubeIndex} \ |= \ (1 \ll i) $$
            If $\text{cubeIndex} == 0$ or $\text{cubeIndex} == 255$, return (no intersection).
        4.  **Generate Vertex List:**
            Initialize `vertexList` array of size 12.
            For each edge $e$ from 0 to 11:
            If $(\text{edgeTable}[\text{cubeIndex}] \ \& \ (1 \ll e)) \neq 0$:
                Get edge endpoints $c_1, c_2$ from `EdgeIndexTable`.
                Get densities $d_1, d_2$.
                Get positions $p_1, p_2$.
                Interpolate vertex:
                $$ v = p_1 + \frac{\text{isolevel} - d_1}{d_2 - d_1} (p_2 - p_1) $$
                $$ \text{vertexList}[e] = v $$
        5.  **Extract Triangles:**
            Get triangle table row for `cubeIndex`.
            Iterate through triplets $(a, b, c)$ until terminator `-1`.
            For each triangle:
                Get atomic vertex index: $v_{idx} = \text{counter.Increment()}$.
                Set $\text{vertices}[v_{idx} + 0] = \text{vertexList}[a]$.
                Set $\text{vertices}[v_{idx} + 1] = \text{vertexList}[b]$.
                Set $\text{vertices}[v_{idx} + 2] = \text{vertexList}[c]$.
                Set $\text{triangles}[v_{idx} + 0] = v_{idx}$.
                Set $\text{triangles}[v_{idx} + 1] = v_{idx} + 1$.
                Set $\text{triangles}[v_{idx} + 2] = v_{idx} + 2$.

4.  **Apply Mesh Data:**
    *   On job completion, calculate `vertexCount = counter.Count * 3`.
    *   Set mesh bounds to size $\times$ size $\times$ size centered at $(size/2, size/2, size/2)$.
    *   Call `SetVertexBufferParams` and `SetVertexBufferData` with `MeshUpdateFlags.DontValidateIndices`.
    *   Call `SetIndexBufferParams` and `SetIndexBufferData`.
    *   Call `SetSubMesh`.
    *   Call `RecalculateNormals()`.

5.  **Bake Collider:**
    *   Schedule `ChunkColliderBakeJob` on a separate thread.
    *   When complete, apply via `MeshCollider.sharedMesh = myMesh`.

6.  **Neighbor Fixing (Edge Case):**
    *   If modifying density at a chunk boundary, check neighbors in $-x, -y, -z$ directions.
    *   If a neighbor chunk is not visible/processed, its border densities may remain inconsistent, causing visual artifacts. The implementation sets border densities to 1.0f if neighbors are out of bounds or not updated [S1, unofficial].

**Parameters:**
*   `isolevel`: Fixed at 0.0. Densities $< 0$ are solid, $> 0$ are air [S1, unofficial].
*   `size`: Chunk grid dimension in voxels. Buffers allocated as $(size^3) \times 3 \times 5$ for triangles/vertices [S1, unofficial].
*   `deformRadius`: Radius for terrain modification [S1, unofficial].
*   `newDensityValue`: 1.0 for cutting, -1.0 for filling [S1, unofficial].

**Pitfalls:**
*   **Neighbor Inconsistency:** If a chunk is not visible, neighboring chunk borders do not get fixed, leaving weird strips [S1, unofficial].
*   **Pooled Chunk Lag:** Pooled chunks must wait for all jobs to complete before reuse, causing significant lag at render distances above 500 [S1, unofficial].
*   **Build Crashes:** Builds may randomly crash, though the editor runs without issues [S1, unofficial].

## Subtopics

### Ball-shaped Terrain Deformation
Modifies the density field of affected chunks to create spherical cutouts or fills, triggering a mesh regeneration [S1, unofficial]. It uses a raycast from the camera to determine the hit point, calculates the affected chunk coordinates, and iterates over a 3D grid of chunks centered on the hit point. For each chunk, it schedules a `ChunkExplodeJob` to modify the `noiseMap` within the deformation radius, followed by a dependent `MarchingJob` to regenerate the mesh [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk. Parallelized via Job System.
*   **Memory Complexity:** $O(N)$ for density map and mesh buffers.
*   **Performance:** Terrain destruction is very fast due to separate-thread collider baking [S1, unofficial]. Infinite terrain generation is slower than desired due to pooled chunk job waiting [S1, unofficial].
*   **Scaling:** Render distance above 500 causes significant lag due to full GameObject overhead and job synchronization [S1, unofficial].
*   **Strengths:** Real-time destruction, infinite terrain, parallel execution.
*   **Limitations:** High draw calls for full GameObjects, potential for visual artifacts at chunk boundaries if neighbors are not updated, build instability.

## Background
This method is an implementation of destructible infinite marching cubes terrain made in Unity 2019.3.14f1 [S1, unofficial]. The code is partially derived from Eldemarkki's Marching-Cubes-Terrain repository [S1, unofficial]. It is used for custom infinite terrain projects with real-time modification capabilities [S1, unofficial].

## Sources
[S1] Fobri/Fast-Unity-Marching-Cubes, https://github.com/Fobri/Fast-Unity-Marching-Cubes, fetched 2026-10-26 (unofficial)
