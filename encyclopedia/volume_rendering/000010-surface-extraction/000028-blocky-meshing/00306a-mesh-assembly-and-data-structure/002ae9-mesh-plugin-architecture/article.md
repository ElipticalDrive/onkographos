# Mesh Plugin Architecture
Also called: IMeshingJob
## Overview
The Mesh Plugin Architecture decouples meshing algorithms from the chunk assembly pipeline by defining a standard interface (`IMeshingJob`) for pluggable surface extraction methods [S1, unofficial]. It allows the engine to switch between blocky face-culling (e.g., Greedy Meshing) and smooth terrain generation (e.g., Marching Cubes) without modifying the assembly logic, ensuring consistent output formats for the downstream buffer allocator [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
*   Voxel data: A 3D grid of voxel structures (solid, smooth, or empty).
*   Chunk metadata: Position, zoom level, and bounds.

**Outputs:**
*   Vertex buffer: Contiguous array of positions.
*   Index buffer: Contiguous array of indices.
*   Attribute buffers: Normals, Block IDs, Ambient Occlusion (AO), and Lighting data.

### Data Structures
The interface requires implementations to produce buffers compatible with the [Mesh Assembly and Data Structure](6268-mesh-assembly-and-data-structure.md). Specifically:
*   **Positions:** $N \times 4 \times 3$ floats (4 vertices per quad, 3 components).
*   **Normals:** $N \times 4 \times 3$ signed bytes (int8).
*   **Indices:** $N \times 6$ unsigned integers (2 triangles per quad).
*   **Attributes:** $N \times 4$ elements per attribute (Block ID, AO, Lights) [S2, unofficial].

### Interface Definition
The core contract is defined by the `IMeshingJob` interface [S1, unofficial]. Implementations must provide a method that accepts voxel data and returns the structured buffers.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class IMeshingJob:
    """Interface for pluggable meshing algorithms."""
    
    def execute(self, voxel_data: VoxelGrid, chunk_metadata: ChunkMetadata) -> MeshOutput:
        """
        Generates mesh buffers from voxel data.
        
        Args:
            voxel_data: The voxel grid to process.
            chunk_metadata: Metadata including position and zoom level.
            
        Returns:
            MeshOutput containing vertex, index, and attribute buffers.
        """
        # Implementation specific to the algorithm (e.g., Greedy, Marching Cubes)
        # Must adhere to the buffer layout defined in Mesh Assembly and Data Structure
        raise NotImplementedError

class MeshOutput:
    """Container for mesh data compatible with the assembly phase."""
    positions: list[float]      # N * 4 * 3 floats
    normals: list[int8]         # N * 4 * 3 signed bytes
    indices: list[uint]         # N * 6 unsigned integers
    attributes: dict[str, list] # Block ID, AO, Lights, etc.
```

### Implementation Examples
1.  **GreedyMeshingJob:** Optimized for blocky voxels. Performs face culling and merges adjacent coplanar quads. It is burst-compiled and generates zero garbage collection (GC) pressure [S1, unofficial].
2.  **MarchingCubesJob:** A simplified prototype for smooth terrain using density fields. Also burst-compiled with zero GC. Full lookup tables are noted for future implementation [S1, unofficial].

### Parameters and Edge Cases
*   **Precision:** Half-precision density fields can save 50% memory (68KB vs 128KB per 32³ chunk) [S1, unofficial].
*   **Job Scheduling:** The `ChunkManager` handles automatic job scheduling and smart mesher selection based on voxel type (blocky vs. smooth) [S1, unofficial].
*   **Coordinate Conversion:** The architecture handles world/local coordinate conversion internally within the manager, abstracting this from the plugin [S1, unofficial].

## Operating characteristics
*   **Performance:** Implementations are designed for burst compilation and zero GC allocation to maintain high frame rates in dynamic voxel worlds [S1, unofficial].
*   **Memory:** Using half-precision for density fields reduces memory footprint by half [S1, unofficial].
*   **Extensibility:** New algorithms can be added by implementing `IMeshingJob` without altering the assembly pipeline [S1, unofficial].
*   **Limitations:** The current Marching Cubes implementation is a simplified prototype; full lookup tables are not yet implemented [S1, unofficial].

## Background
The architecture was introduced in the "StraightFour" voxel engine project to support both generic blocky voxels and smooth terrain in a unified pipeline [S1, unofficial]. It separates the concerns of surface extraction from mesh assembly, allowing for specialized algorithms like greedy meshing for performance and marching cubes for smoothness [S1, unofficial]. The `MeshStatus` enum and `MeshInfo` structures in the Infinigen project further refine how these generated meshes are tracked and stored [S2, unofficial].

## Subtopics

### Greedy Meshing
A blocky voxel optimization technique that merges adjacent coplanar faces into larger quads. Chosen for performance in blocky worlds. [Greedy Meshing](greedy-meshing.md)

### Marching Cubes
A surface extraction algorithm for smooth terrain from density fields. Chosen for organic shapes. [Marching Cubes](marching-cubes.md)

## Sources
[S1] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
