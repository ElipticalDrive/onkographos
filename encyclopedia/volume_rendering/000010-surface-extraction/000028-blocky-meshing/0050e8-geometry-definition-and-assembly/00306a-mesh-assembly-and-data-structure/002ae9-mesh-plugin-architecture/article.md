# Mesh Plugin Architecture

Also called: IMeshingJob

## Overview
The Mesh Plugin Architecture provides an interface for pluggable meshing algorithms, allowing for extensibility within the geometry generation pipeline [S1, unofficial]. It decouples the meshing strategy from the chunk management system, enabling the selection of algorithms based on voxel type (e.g., blocky vs. smooth) [S1, unofficial]. This approach sets it apart from monolithic assembly methods by focusing on the algorithmic abstraction of surface extraction rather than buffer aggregation.

## How it works
The architecture defines a contract (`IMeshingJob`) that specific meshing implementations must satisfy. The `ChunkManager` acts as the orchestrator, handling spatial hashing, job scheduling, and smart mesher selection [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Voxel data (solid or smooth density) and chunk coordinates.
*   **Output:** Raw geometry data (vertices, indices, normals) prepared for the assembly phase.

**Data Structures:**
*   `IMeshingJob`: An interface defining the method signature for mesh generation.
*   `MeshStatus`: An enum tracking the state of a chunk's mesh:
    *   `Meshing`: Generation in progress.
    *   `Meshed(Arc<MeshInfo>)`: Generation complete, holding a reference to the mesh data.
    *   `Empty`: No mesh data exists.
*   `MeshInfo`: Contains an optional opaque mesh and a vector of translucent meshes [S2, unofficial].

**Numbered Steps:**
1.  **Voxel Registration:** The user registers voxels via `ChunkManager.SetVoxel`, specifying position, type (solid/smooth), and material ID [S1, unofficial].
2.  **Smart Mesher Selection:** The `ChunkManager` determines the appropriate `IMeshingJob` implementation based on the voxel type [S1, unofficial].
    *   For blocky voxels, it selects `GreedyMeshingJob` (face culling) [S1, unofficial].
    *   For smooth terrain, it selects `MarchingCubesJob` (simplified MC) [S1, unofficial].
3.  **Job Execution:** The selected job runs, typically burst-compiled for performance with zero garbage collection [S1, unofficial].
    *   `GreedyMeshingJob` performs face culling for blocky voxels.
    *   `MarchingCubesJob` implements a simplified Marching Cubes algorithm, noting full lookup tables for future implementation [S1, unofficial].
4.  **Status Update:** The `MeshStatus` of the chunk is updated to `Meshing` during execution and `Meshed` upon completion, storing the result in a hash map keyed by `(ChunkPosition, ZoomLevel)` [S2, unofficial].

**Parameters:**
*   **Precision:** Half-precision density is used to save 50% memory (68KB vs 128KB per 32³ chunk) [S1, unofficial].
*   **Optimization:** Jobs are burst-compiled to ensure zero GC overhead [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Lookup Tables:** The simplified MC prototype notes that full lookup tables are not yet implemented, which may limit correctness or performance for complex smooth terrain [S1, unofficial].
*   **Job Scheduling:** Sequential job processing is mentioned as a potential optimization or constraint, though details are cut off [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Interface definition
class IMeshingJob:
    def generate(self, chunk_data: VoxelChunk) -> MeshData:
        """Generates mesh data from voxel chunk."""
        pass

# Implementation: Greedy Meshing for Blocky Voxels
class GreedyMeshingJob(IMeshingJob):
    def generate(self, chunk_data: VoxelChunk) -> MeshData:
        # Perform face culling for blocky voxels
        # Burst-compiled, zero GC
        return MeshData.from_face_culling(chunk_data)

# Implementation: Marching Cubes for Smooth Terrain
class MarchingCubesJob(IMeshingJob):
    def generate(self, chunk_data: VoxelChunk) -> MeshData:
        # Simplified MC prototype
        # Full lookup tables noted for future implementation
        return MeshData.from_marching_cubes(chunk_data)

# Orchestrator
class ChunkManager:
    def __init__(self):
        self.mesh_status_map = {} # Hash map of (ChunkPosition, ZoomLevel) to MeshStatus
        self.chunk_material = None

    def set_voxel(self, position: int3, voxel_data: VoxelData):
        # Register voxel
        pass

    def select_mesher(self, voxel_data: VoxelData) -> IMeshingJob:
        if voxel_data.is_blocky():
            return GreedyMeshingJob()
        elif voxel_data.is_smooth():
            return MarchingCubesJob()
        else:
            raise ValueError("Unknown voxel type")

    def process_chunk(self, chunk_position: int3):
        # Smart mesher selection
        mesher = self.select_mesher(self.get_voxel_data(chunk_position))
        
        # Update status to Meshing
        self.mesh_status_map[(chunk_position, 0)] = MeshStatus.Meshing
        
        # Execute job
        mesh_data = mesher.generate(self.get_chunk_data(chunk_position))
        
        # Update status to Meshed
        self.mesh_status_map[(chunk_position, 0)] = MeshStatus.Meshed(Arc(MeshInfo(mesh_data)))
```

## Operating characteristics
*   **Performance:** Jobs are burst-compiled and designed for zero garbage collection, optimizing CPU usage in managed environments like Unity [S1, unofficial].
*   **Memory:** Half-precision density reduces memory footprint by 50% for 32³ chunks [S1, unofficial].
*   **Scalability:** The architecture supports automatic job scheduling and spatial hashing, allowing for efficient handling of large worlds [S1, unofficial].
*   **Limitations:** The simplified MC implementation lacks full lookup tables, which may impact the fidelity or speed of smooth terrain generation [S1, unofficial].

## Background
The Mesh Plugin Architecture was introduced as part of the foundational voxel engine development for the StraightFour project [S1, unofficial]. It aims to provide a generic, extensible framework for both terrain and smooth voxel support, allowing developers to swap meshing algorithms without modifying the core chunk management logic [S1, unofficial].

## Sources
[S1] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
