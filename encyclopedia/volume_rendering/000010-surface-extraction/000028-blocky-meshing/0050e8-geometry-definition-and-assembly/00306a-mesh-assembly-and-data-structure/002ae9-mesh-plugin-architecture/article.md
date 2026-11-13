# Mesh Plugin Architecture

Also called: IMeshingJob

## Overview
The Mesh Plugin Architecture provides an interface for pluggable meshing algorithms, allowing for extensibility within the chunk meshing pipeline [S1, unofficial]. It decouples the mesh generation logic from the chunk management system, enabling the selection of different algorithms (e.g., greedy meshing for blocky voxels vs. marching cubes for smooth terrain) based on voxel type [S1, unofficial]. This approach complements the parent [Mesh Assembly and Data Structure](6268-mesh-assembly-and-data-structure.md) by defining *how* geometry is created before it is assembled into contiguous buffers.

## How it works
The architecture is defined by a single interface, `IMeshingJob`, which standardizes the contract for meshing algorithms [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A chunk of voxel data (e.g., `VoxelData`), which may contain blocky or smooth density information.
*   **Output:** A mesh representation (e.g., `MeshInfo`), which contains an optional opaque mesh and a vector of translucent meshes [S2, unofficial].

**Data Structures:**
The system tracks the state of meshing for each chunk using a hash map structure:
*   **Key:** `(ChunkPosition, ZoomLevel)`
*   **Value:** `MeshStatus`
*   **MeshStatus Enum:**
    *   `Meshing`: The algorithm is currently running.
    *   `Meshed(Arc<MeshInfo>)`: Meshing is complete; contains the resulting geometry.
    *   `Empty`: No mesh data exists.

**Steps:**
1.  **Voxel Registration:** The user registers voxels with specific types (e.g., `VoxelData.CreateSolid` for blocky, `VoxelData.CreateSmooth` for density) [S1, unofficial].
2.  **Smart Mesher Selection:** The `ChunkManager` analyzes the voxel type and selects the appropriate implementation of `IMeshingJob` [S1, unofficial].
    *   For blocky voxels, it selects `GreedyMeshingJob` (face culling for blocky voxels) [S1, unofficial].
    *   For smooth terrain, it selects `MarchingCubesJob` (simplified MC prototype) [S1, unofficial].
3.  **Job Execution:** The selected job runs, typically burst-compiled for performance with zero garbage collection [S1, unofficial].
4.  **Status Update:** Upon completion, the `MeshStatus` for the chunk is updated to `Meshed` with the generated `MeshInfo` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class IMeshingJob:
    """Interface for pluggable meshing algorithms."""
    def execute(self, chunk_data: VoxelData) -> MeshInfo:
        """
        Generates mesh info from voxel data.
        Returns: MeshInfo containing opaque and translucent meshes.
        """
        raise NotImplementedError

class GreedyMeshingJob(IMeshingJob):
    """Face culling for blocky voxels. Burst-compiled, zero GC."""
    def execute(self, chunk_data: VoxelData) -> MeshInfo:
        # Perform face culling and greedy meshing
        opaque_mesh = perform_greedy_mesh(chunk_data)
        return MeshInfo(opaque=opaque_mesh, translucent=[])

class MarchingCubesJob(IMeshingJob):
    """Simplified MC prototype for smooth terrain. Burst-compiled, zero GC."""
    def execute(self, chunk_data: VoxelData) -> MeshInfo:
        # Perform marching cubes on density field
        # Full lookup tables noted for future implementation [S1, unofficial]
        mesh = perform_marching_cubes(chunk_data.density_field)
        return MeshInfo(opaque=mesh, translucent=[])

class ChunkManager:
    def __init__(self):
        self.mesh_status_map = {} # Hash map of (ChunkPosition, ZoomLevel) to MeshStatus
        self.mesher_map = {
            "blocky": GreedyMeshingJob(),
            "smooth": MarchingCubesJob()
        }

    def set_voxel(self, position: int3, voxel_data: VoxelData):
        # Update voxel data
        pass

    def process_chunk(self, position: int3, zoom_level: int):
        # Smart mesher selection based on voxel type
        voxel_type = self.get_voxel_type(position)
        job = self.mesher_map[voxel_type]
        
        # Execute job
        mesh_info = job.execute(self.get_chunk_data(position))
        
        # Update status
        self.mesh_status_map[(position, zoom_level)] = MeshStatus.Meshed(mesh_info)
```

**Parameters:**
*   **Precision:** Half precision density is used to save 50% memory (68KB vs 128KB per 32³ chunk) [S1, unofficial].

**Edge Cases:**
*   **Lookup Tables:** The MarchingCubesJob implementation notes that full lookup tables are reserved for future implementation, implying the current version may use simplified or hardcoded logic [S1, unofficial].

## Subtopics

### Greedy Meshing
Optimizes blocky voxel meshes by culling internal faces and merging adjacent quads. Chosen for solid, non-transparent blocky voxels. [GreedyMeshingJob](4100-chunk-mesh-assembly.md)

### Marching Cubes
Generates smooth geometry from density fields. Chosen for smooth terrain or fluid-like voxels. [MarchingCubesJob](4100-chunk-mesh-assembly.md)

## Operating characteristics
*   **Performance:** Jobs are burst-compiled and designed for zero garbage collection (GC) to maintain high performance in real-time applications [S1, unofficial].
*   **Memory:** Using half-precision density reduces memory usage by 50% compared to full precision (68KB vs 128KB per 32³ chunk) [S1, unofficial].
*   **Scalability:** The architecture supports automatic job scheduling and spatial hashing via `ChunkManager`, allowing for efficient handling of large worlds [S1, unofficial].
*   **Limitations:** The current MarchingCubes implementation is a prototype and lacks full lookup tables, which may limit its accuracy or performance compared to a full implementation [S1, unofficial].

## Background
The Mesh Plugin Architecture was introduced in the "StraightFour" voxel engine project to support both generic, terrain, and smooth voxel rendering [S1, unofficial]. It was developed to allow flexible meshing strategies without modifying the core chunk management logic. The `IMeshingJob` interface was added in a pull request to the StraightFour repository [S1, unofficial]. The `MeshStatus` and `MeshInfo` structures are part of the Infinigen project, which uses similar concepts for managing mesh states in a Rust-based environment [S2, unofficial].

## Sources
[S1] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
