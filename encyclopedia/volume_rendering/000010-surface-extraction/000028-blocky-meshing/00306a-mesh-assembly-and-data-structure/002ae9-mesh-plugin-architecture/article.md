# Mesh Plugin Architecture

Also called: IMeshingJob

## Overview
The Mesh Plugin Architecture provides an interface for pluggable meshing algorithms, allowing for extensibility within the meshing pipeline [S1, unofficial]. It decouples the mesh generation logic from the chunk management system, enabling the selection of specific algorithms (e.g., greedy meshing for blocky voxels vs. marching cubes for smooth terrain) based on voxel data types [S1, unofficial]. This approach complements the [Mesh Assembly and Data Structure](6268-mesh-assembly-and-data-structure.md) by defining *how* geometry is created before it is assembled into the final resource stores.

## How it works
The architecture is defined by a single interface, `IMeshingJob`, which standardizes the contract for meshing algorithms [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Voxel data (e.g., `VoxelData` containing material IDs or density fields) and chunk metadata.
*   **Output:** Vertex and index buffers compatible with the [Mesh Assembly and Data Structure](6268-mesh-assembly-and-data-structure.md).

**Data Structures:**
*   `IMeshingJob`: An interface defining the meshing method.
*   `GreedyMeshingJob`: A concrete implementation for face culling of blocky voxels [S1, unofficial].
*   `MarchingCubesJob`: A prototype implementation for smooth terrain using simplified Marching Cubes [S1, unofficial].

**Steps:**
1.  The `ChunkManager` identifies the voxel type (blocky or smooth) [S1, unofficial].
2.  The manager selects the appropriate `IMeshingJob` implementation (e.g., `GreedyMeshingJob` for blocky, `MarchingCubesJob` for smooth) [S1, unofficial].
3.  The selected job executes, processing the chunk's voxel data.
4.  The job generates mesh data, which is then passed to the assembly phase [S1, unofficial].

**Parameters:**
*   **Precision:** Half-precision density is used to save 50% memory (68KB vs 128KB per 32³ chunk) [S1, unofficial].
*   **Compilation:** Jobs are burst-compiled for zero garbage collection (GC) overhead [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode representing the interface and selection logic
# Note: Specific internal implementation of GreedyMeshingJob and MarchingCubesJob is not provided in sources [S1, unofficial]

class IMeshingJob:
    def process_chunk(self, chunk_data: VoxelData) -> MeshData:
        """
        Interface method for pluggable meshing algorithms.
        """
        raise NotImplementedError

class GreedyMeshingJob(IMeshingJob):
    def process_chunk(self, chunk_data: VoxelData) -> MeshData:
        # Face culling for blocky voxels
        # Burst-compiled, zero GC
        return apply_face_culling(chunk_data)

class MarchingCubesJob(IMeshingJob):
    def process_chunk(self, chunk_data: VoxelData) -> MeshData:
        # Simplified MC prototype for smooth terrain
        # Burst-compiled, zero GC
        # Full lookup tables noted for future implementation
        return apply_marching_cubes(chunk_data)

class ChunkManager:
    def __init__(self):
        self.chunk_material = None
        self.meshing_jobs = {
            "blocky": GreedyMeshingJob(),
            "smooth": MarchingCubesJob()
        }

    def set_voxel(self, position: int3, voxel_data: VoxelData):
        """
        Sets a voxel and triggers automatic mesh generation via Jobs.
        """
        # Store voxel data
        self.store_voxel(position, voxel_data)
        
        # Smart mesher selection
        job_type = "blocky" if voxel_data.is_solid() else "smooth"
        job = self.meshing_jobs[job_type]
        
        # Execute job
        mesh_data = job.process_chunk(self.get_chunk_data(position))
        
        # Pass to assembly (see Mesh Assembly and Data Structure)
        self.assemble_mesh(mesh_data)
```

## Operating characteristics
*   **Performance:** Jobs are burst-compiled to ensure zero garbage collection (GC) overhead, critical for real-time performance [S1, unofficial].
*   **Memory:** Using half-precision density reduces memory usage by 50% (68KB vs 128KB per 32³ chunk) [S1, unofficial].
*   **Extensibility:** The interface allows adding new meshing algorithms without modifying the core chunk management logic [S1, unofficial].
*   **Limitations:** The Marching Cubes implementation is noted as a "simplified prototype" with full lookup tables planned for future implementation [S1, unofficial].

## Background
The Mesh Plugin Architecture was introduced in the StraightFour voxel engine project [S1, unofficial]. It was designed to support both generic blocky voxels and smooth terrain within a single engine framework. The architecture leverages job-based systems (likely ECS or similar) to parallelize mesh generation.

## Sources
[S1] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
