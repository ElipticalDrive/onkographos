# Mesh Build Scheduling and Application

Also called: Job Scheduling, MeshBuildScheduler

## Overview
A scheduler that manages the lifecycle of mesh building jobs, handling job submission, completion, and the application of generated mesh data to Unity Mesh objects, including pool management for mesh instances [S1, unofficial]. It distinguishes itself from its parent method by explicitly handling the synchronization between the asynchronous Unity Job System execution and the synchronous Unity Mesh API, ensuring that generated geometry is correctly mapped to persistent mesh objects via a chunk pool [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** `VloxySettings` (configuration), `ChunkManager` (access to voxel data), `ChunkPool` (mesh object management), and a list of chunk positions to build (`jobs`).
*   **Outputs:** Updated Unity `Mesh` objects associated with the chunk positions, ready for rendering.

**Data Structures:**
*   `MeshBuildScheduler`: The main scheduler class.
*   `VertexParams`: A structure containing 6 descriptors defining vertex layout [S1, unofficial].
*   `_Results`: A hash map (dictionary) mapping chunk positions to mesh indices or IDs. Its size is determined by `settings.Chunk.DrawDistance.CubedSize()` [S1, unofficial].
*   `_Jobs`: A `NativeList` storing the positions of chunks to be processed [S1, unofficial].
*   `_MeshDataArray`: A writable array of mesh data allocated via `UnityEngine.Mesh.AllocateWritableMeshData` [S1, unofficial].
*   `_Handle`: A `JobHandle` returned by the job scheduling system [S1, unofficial].

**Parameters:**
*   `DrawDistance`: Determines the capacity of the `_Results` hash map. The size is calculated as `settings.Chunk.DrawDistance.CubedSize()` [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   Initialize `MeshBuildScheduler` with `VloxySettings`, `ChunkManager`, and `ChunkPool` [S1, unofficial].
    *   Allocate `VertexParams` with 6 descriptors [S1, unofficial].
    *   Allocate `_Results` hash map with size `settings.Chunk.DrawDistance.CubedSize()` [S1, unofficial].

2.  **Job Submission (`Start(jobs)`):**
    *   Set `IsReady = false` [S1, unofficial].
    *   Get `ChunkAccessor` from `ChunkManager` for the provided list of jobs [S1, unofficial].
    *   Add job positions to `_Jobs` NativeList [S1, unofficial].
    *   Allocate writable mesh data: `_MeshDataArray = UnityEngine.Mesh.AllocateWritableMeshData(_Jobs.Length)` [S1, unofficial].
    *   Construct `MeshBuildJob` with the following fields:
        *   `Accessor`: The `ChunkAccessor` [S1, unofficial].
        *   `ChunkSize`: The size of a chunk [S1, unofficial].
        *   `Jobs`: The `_Jobs` NativeList [S1, unofficial].
        *   `VertexParams`: The `VertexParams` structure [S1, unofficial].
        *   `MeshDataArray`: The `_MeshDataArray` [S1, unofficial].
        *   `Results`: A parallel writer for the `_Results` hash map (`_Results.AsParallelWriter()`) [S1, unofficial].
    *   Schedule the job: `_Handle = job.Schedule(_Jobs.Length, 1)` [S1, unofficial].

3.  **Completion and Application (`Complete()`):**
    *   Wait for job completion: `_Handle.Complete()` [S1, unofficial].
    *   Create an array of Unity Mesh objects: `meshes = new UnityEngine.Mesh[_Jobs.Length]` [S1, unofficial].
    *   Iterate through job indices:
        *   Get chunk position from `_Jobs[index]` [S1, unofficial].
        *   Check if the chunk was remeshed: If `_ChunkManager.ReMeshedChunk(position)` is true, get the mesh from the pool using `_ChunkPool.Get(position).Mesh` [S1, unofficial].
        *   If not remeshed (new chunk), claim a mesh from the pool using `_ChunkPool.Claim(position).Mesh` [S1, unofficial].
        *   Assign the mesh to the array: `meshes[_Results[position]] = ...` [S1, unofficial].
    *   Apply mesh data to Unity meshes: `UnityEngine.Mesh.ApplyAndDisposeWritableMeshData(_MeshDataArray, meshes, MeshUpdateFlags.DontRecalculateBounds)` [S1, unofficial].
    *   Recalculate bounds for all meshes: `meshes[index].RecalculateBounds()` for each index [S1, unofficial].
    *   Clear `_Results` and `_Jobs` [S1, unofficial].
    *   Set `IsReady = true` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MeshBuildScheduler:
    def __init__(self, settings, chunk_manager, chunk_pool):
        self._settings = settings
        self._chunk_manager = chunk_manager
        self._chunk_pool = chunk_pool
        self._vertex_params = VertexParams(6 descriptors)
        # Size based on draw distance cubed
        self._results = HashMap(size=settings.chunk.draw_distance.cubed_size())
        self._jobs = NativeList()
        self._mesh_data_array = None
        self._handle = None
        self._is_ready = False

    def start(self, jobs):
        self._is_ready = False
        self._chunk_accessor = self._chunk_manager.get_accessor(jobs)
        self._jobs.clear()
        for pos in jobs:
            self._jobs.add(pos)
        
        # Allocate writable mesh data for all jobs
        self._mesh_data_array = UnityEngine.Mesh.allocate_writable_mesh_data(self._jobs.length)
        
        job = MeshBuildJob(
            accessor=self._chunk_accessor,
            chunk_size=self._chunk_size,
            jobs=self._jobs,
            vertex_params=self._vertex_params,
            mesh_data_array=self._mesh_data_array,
            results=self._results.as_parallel_writer()
        )
        
        # Schedule job with 1 chunk per job
        self._handle = job.schedule(self._jobs.length, 1)

    def complete(self):
        # Wait for job completion
        self._handle.complete()
        
        meshes = new UnityEngine.Mesh[self._jobs.length]
        
        for index in range(self._jobs.length):
            position = self._jobs[index]
            result_index = self._results[position]
            
            if self._chunk_manager.remeshed_chunk(position):
                # Reuse existing mesh object
                meshes[result_index] = self._chunk_pool.get(position).mesh
            else:
                # Claim new mesh object from pool
                meshes[result_index] = self._chunk_pool.claim(position).mesh
        
        # Apply data and dispose the writable array
        UnityEngine.Mesh.apply_and_dispose_writable_mesh_data(
            self._mesh_data_array, 
            meshes, 
            MeshUpdateFlags.DontRecalculateBounds
        )
        
        # Recalculate bounds for each mesh
        for index in range(meshes.length):
            meshes[index].recalculate_bounds()
            
        self._results.clear()
        self._jobs.clear()
        self._is_ready = True
```

**Edge Cases and Pitfalls:**
*   **Mesh Pooling:** The scheduler must distinguish between remeshed chunks and new chunks. Using `_ChunkPool.Get(position)` for remeshed chunks reuses existing mesh objects, while `_ChunkPool.Claim(position)` is used for new chunks to ensure efficient memory management [S1, unofficial]. Failure to distinguish may lead to memory leaks or incorrect mesh references.
*   **Bounds Recalculation:** Bounds are recalculated after applying mesh data because `MeshUpdateFlags.DontRecalculateBounds` is used during application to save performance [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the mesh building job itself (see [Greedy Meshing with Unity MeshData API](21929-greedy-meshing-with-unity-meshdata-api.md)) and the synchronization cost of `JobHandle.Complete()`.
*   **Memory Complexity:** Proportional to the number of jobs and the draw distance cubed (for the `_Results` hash map) [S1, unofficial].
*   **Strengths:** Efficient memory management through chunk pooling; avoids CPU-GPU synchronization overhead by using the MeshData API [S1, unofficial].
*   **Limitations:** Requires Unity's Job System and MeshData API; specific to Unity engine [S1, unofficial].
*   **When to use:** When building voxel meshes in Unity and needing to manage the lifecycle of mesh objects efficiently [S1, unofficial].
*   **When not to use:** In non-Unity environments or when mesh building is not parallelized [S1, unofficial].

## Background
This method is part of the VloxyEngine project, an unofficial voxel engine implementation for Unity [S1, unofficial]. It builds upon the greedy meshing algorithm and Unity's MeshData API to optimize mesh generation [S1, unofficial].

## Sources
[S1] BLaZeKiLL/VloxyEngine, https://github.com/BLaZeKiLL/VloxyEngine, fetched 2026-10-26 (unofficial)
