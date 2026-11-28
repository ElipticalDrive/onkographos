# Marching Cubes Seam In-Place Reading

Also called: In-Place Seam Reading, MC Seam Direct Snapshot Access

## Overview
Marching Cubes Seam In-Place Reading is an optimization for CPU-based Marching Cubes seam meshing that eliminates the memory bandwidth overhead of copying neighbor voxel data into synthetic request structures [S1, unofficial]. Instead of duplicating volume snapshots for each seam job, the method reads neighbor data directly from the participants' existing snapshots via read-only pointers [S1, unofficial]. This approach reduces seam worker execution time by approximately 30x while producing bit-identical geometry to the original copy-based method [S1, unofficial]. It is distinct from general memory optimization techniques [12239-memory-and-performance-optimization.md] as it specifically targets the seam closure phase of the Marching Cubes algorithm [30-isosurface-extraction.md].

## How it works
The method operates on the CPU Marching Cubes mesher during the seam closure phase, where mesh seams between adjacent chunks or LOD levels must be closed to ensure manifold continuity [S1, unofficial].

### Inputs and Outputs
- **Input**: A set of "participant" frames, each containing a voxel volume snapshot (e.g., $64^3$ grid, ~1 MB per frame) [S1, unofficial].
- **Input**: A "seam frame" requiring mesh closure with its neighbors [S1, unofficial].
- **Output**: A seam mesh that is bit-identical to the geometry produced by the original copy-based method [S1, unofficial].

### Data Structures
The core data structure is `FVoxelMeshingRequest::VolumeViews` [S1, unofficial].
- **Layout**: A structure containing 27 pointers, one for each neighbor direction in a $3 \times 3 \times 3$ neighborhood relative to the seam frame [S1, unofficial].
- **Access**: These pointers are read-only by the CPU Marching Cubes mesher [S1, unofficial].
- **Content**: Each pointer references the corresponding participant's voxel snapshot directly [S1, unofficial].

### Parameters
- `voxel.Seam.MCReadInPlace`: A boolean flag [S1, unofficial].
  - `1`: Enables in-place reading (uses `VolumeViews`) [S1, unofficial].
  - `0`: Disables in-place reading (uses the original copy-based method) [S1, unofficial].

### Steps
1. **Initialization**: When a seam frame is created, instead of allocating synthetic request structures and copying volume data, the system populates `FVoxelMeshingRequest::VolumeViews` with pointers to the neighbor participants' snapshots [S1, unofficial].
2. **Validation**: The mesher checks data validity using `IsVoxelHeld` and `HasRequiredNeighborData` to ensure the referenced snapshots are valid and accessible [S1, unofficial].
3. **Voxel Access**: When the mesher needs neighbor data, it calls `GetVoxelAt` [S1, unofficial].
   - This function reads the neighbor voxel directly from the snapshot pointed to by `VolumeViews` [S1, unofficial].
   - It replicates the exact behavior of the original copy method:
     - **Face Planes**: Reads neighbor planes up to `NeighborPlaneDepth`, applying `NeighborPlaneVoxel`'s clamp logic [S1, unofficial].
     - **Edge Strips**: Reads the necessary edge strip data [S1, unofficial].
     - **Corners**: Reads the corner data, which is one voxel deep [S1, unofficial].
4. **Meshing**: The Marching Cubes algorithm proceeds using these in-place reads to generate the seam mesh [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structure Definition
class FVoxelMeshingRequest:
    # 27 pointers for neighbor directions (3x3x3 neighborhood)
    # Read-only by CPU MC mesher
    VolumeViews: array[27] of Pointer to VoxelSnapshot

# Helper Functions (from original method, behavior preserved)
def IsVoxelHeld(snapshot):
    # Check if snapshot is valid and accessible
    return snapshot is not None

def HasRequiredNeighborData(request):
    # Test if all required neighbor views are valid
    for view in request.VolumeViews:
        if not IsVoxelHeld(view):
            return False
    return True

def GetVoxelAt(snapshot, direction, x, y, z):
    """
    Reads a neighbor voxel exactly as the copy held it.
    direction: Index into VolumeViews (0-26)
    x, y, z: Coordinates within the neighbor's local space
    """
    snapshot = request.VolumeViews[direction]
    
    # Face planes: handle NeighborPlaneDepth and NeighborPlaneVoxel's clamp
    if is_face_plane_read(direction):
        depth = NeighborPlaneDepth
        # Clamp logic applied as in original copy method
        voxel = snapshot.get_clamped_voxel(x, y, z, depth)
        return voxel
    
    # Edge strips: read as held by copy
    elif is_edge_strip_read(direction):
        voxel = snapshot.get_voxel(x, y, z)
        return voxel
    
    # Corner: one voxel deep
    elif is_corner_read(direction):
        voxel = snapshot.get_voxel(x, y, z)
        return voxel
    
    # Default fallback
    else:
        return snapshot.get_voxel(x, y, z)

def ProcessSeamFrame(request):
    """
    Main entry point for seam processing with in-place reading.
    """
    # Validate neighbor data availability
    if not HasRequiredNeighborData(request):
        raise Error("Missing required neighbor data")
    
    # Iterate over seam cells
    for cell in request.seam_cells:
        # For each cell, access neighbor data via VolumeViews
        # instead of copying from synthetic requests
        neighbor_data = {}
        for direction in range(27):
            neighbor_data[direction] = GetVoxelAt(request, direction, cell.x, cell.y, cell.z)
        
        # Marching Cubes algorithm uses neighbor_data to generate seam mesh
        generate_marching_cubes_mesh(cell, neighbor_data)
```

### Edge Cases and Pitfalls
- **Dual Contouring Grids**: The deep grids used by Dual Contouring are not read by Marching Cubes [S1, unofficial]. Therefore, they do not need to be included in the neighbor data gathering or in-place views [S1, unofficial].
- **GPU Marching Cubes**: This optimization only applies to the CPU Marching Cubes mesher; the GPU Marching Cubes implementation has no seam path and is unaffected [S1, unofficial].
- **Dual Contouring and Cubic Code**: These components are untouched by this optimization [S1, unofficial].
- **Precision**: The in-place method produces bit-identical seam geometry to the copy method, verified across 84 test cases [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The optimization reduces seam worker execution time by approximately 30x [S1, unofficial].
  - On `Review_Planet_L` MC seam worker: Original method took 34.1–37.5 s (5.98–6.57 ms per job); in-place method took 1.0–1.2 s (0.18–0.21 ms per job) [S1, unofficial].
  - On `VoxelDemo` MC seam worker: Original method took 44.1–46.0 s (8.34–8.69 ms per job); in-place method took 0.94–0.96 s (0.18 ms per job) [S1, unofficial].
- **Memory Complexity**: Eliminates the memory bandwidth and copy overhead associated with duplicating volume snapshots for each seam job [S1, unofficial]. The original method required copying two volumes for face jobs, four for edge jobs, and eight for corner jobs to mesh one cell layer [S1, unofficial].
- **Strengths**: Significant performance improvement in seam closure; bit-identical output; no changes to meshing logic required.
- **Limitations**: Only applicable to CPU Marching Cubes; requires valid neighbor snapshots to be accessible via pointers.
- **When to Use**: When CPU Marching Cubes seam closure is a performance bottleneck and neighbor snapshots are readily available in memory.
- **When Not to Use**: When using GPU Marching Cubes (no seam path); when neighbor snapshots are not accessible or cannot be referenced via pointers.

## Background
This optimization was introduced in a GitHub Pull Request (#149) to the `VoxelWorlds` repository by draquel in October 2026 [S1, unofficial]. It addresses the high cost of Marching Cubes seam jobs, which were previously about 23 times slower than Dual Contouring jobs [S1, unofficial]. The original method involved building a synthetic request per Marching Cubes seam frame, copying the frame's whole volume (1 MB at $64^3$), and gathering neighbors' planes, edge grids, and corner boxes [S1, unofficial]. The in-place reading method replaces this with direct snapshot access via `FVoxelMeshingRequest::VolumeViews` [S1, unofficial].

## Sources
[S1] G63: Marching Cubes seam jobs read their participants in place, https://github.com/draquel/VoxelWorlds/pull/149, fetched 2026-10-27 (unofficial)
