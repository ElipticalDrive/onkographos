# Sparse VDB SDF Generation

Also called: OpenVDB SDF, meshToDistanceVdbVolume

## Overview
A method that converts a triangle mesh into a sparse, tree-structured signed or unsigned distance grid using OpenVDB, storing only active voxels near the surface for memory efficiency [S1]. It differs from dense voxelization by dynamically allocating memory only for the narrow band around the mesh, making it suitable for large or complex models where ray-marched rendering or fast implicit queries are required [S1].

## How it works
**Inputs and Outputs:**
- Input: A triangle mesh file (e.g., STL format) [S1].
- Output: A `VdbVolume` object containing the sparse distance field [S1]. Optionally, a reconstructed mesh can be extracted from the volume [S1].

**Data Structures:**
- `MeshToVolumeParams`: Configuration structure containing `surfaceOffset` (integer for active voxel layers), `type` (enum for Signed or Unsigned distance), `voxelSize` (`Vector3f` for physical voxel dimensions), and `outXf` (`AffineXf3f` for spatial shifting) [S1].
- `GridToMeshSettings`: Configuration structure for reconstruction containing `voxelSize` (matching generation dimensions) and `isoValue` (float for the isosurface threshold) [S1].

**Steps:**
1. Load the input mesh from a file [S1].
2. Initialize the `MeshToVolumeParams` configuration object [S1].
3. Set `surfaceOffset` to 3 to define the active voxel band width [S1].
4. Determine `type` by checking if the mesh topology is closed, assigning `Signed` if true and `Unsigned` otherwise [S1].
5. Calculate `voxelSize` by querying `suggestVoxelSize` with a target of 10 million voxels and wrapping the result in a diagonal vector [S1].
6. Initialize `voxelsShift` as an identity transformation matrix and assign it to `params.outXf` [S1].
7. Call `meshToDistanceVdbVolume` with the mesh and parameters to generate the sparse volume [S1].
8. Initialize `GridToMeshSettings` with the calculated `voxelSize` and an isovalue of 0.0 [G].
9. Call `gridToMesh` on the volume data and settings to extract a triangle mesh [S1].
10. Apply the shift transformation to the extracted mesh to restore its original spatial position [S1].

**Formulas:**
- `voxelSize = Vector3f.diagonal(suggestVoxelSize(mesh, 1e7))` [S1].
- `isoValue = 0.0` [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from meshlib import mrmeshpy as mm

# 1. Load mesh
mesh = mm.loadMesh("mesh.stl")

# 2. Initialize parameters
params = mm.MeshToVolumeParams()

# 3. Set surface offset (3 voxel layers)
params.surfaceOffset = 3

# 4. Determine distance type based on topology
if mesh.topology.isClosed():
    params.type = mm.MeshToVolumeParams.Type.Signed
else:
    params.type = mm.MeshToVolumeParams.Type.Unsigned

# 5. Calculate voxel size targeting ~10 million voxels
params.voxelSize = mm.Vector3f.diagonal(mm.suggestVoxelSize(mesh, 1e7))

# 6. Initialize shift transformation
voxelsShift = mm.AffineXf3f()
params.outXf = voxelsShift

# 7. Generate sparse VDB volume
vdbVolume = mm.meshToDistanceVdbVolume(mesh, params)

# --- Optional: Reconstruct Mesh from VDB ---

# 8. Initialize grid-to-mesh settings
gSettings = mm.GridToMeshSettings()
gSettings.voxelSize = params.voxelSize
gSettings.isoValue = 0.0  # Standard SDF isosurface threshold [G]

# 9. Extract mesh from volume data
offMesh = mm.gridToMesh(vdbVolume.data, gSettings)

# 10. Transform output mesh back to original position
offMesh.transform(voxelsShift)

# 11. Save result
mm.saveMesh(offMesh, "out_mesh.stl")
```

**Parameters:**
- `surfaceOffset`: Integer specifying the number of voxel layers around the surface to mark as active [S1].
- `type`: Enum indicating `Signed` for closed meshes or `Unsigned` for open meshes [S1].
- `voxelSize`: `Vector3f` defining the physical size of each voxel, calculated to target approximately 10 million voxels [S1].
- `outXf`: `AffineXf3f` transformation matrix applied to shift output voxels relative to the original mesh [S1].

**Edge Cases and Pitfalls:**
- The input mesh must be topologically closed for `Signed` distance calculation, otherwise `Unsigned` must be selected [S1].
- The `isoValue` for reconstruction is specified in voxel units, not world units [S1].
- The shift matrix must be applied to the reconstructed mesh to align it with the original coordinate space [S1].

## Subtopics

### [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md)
Converts explicit surfaces to Truncated Signed Distance Fields. Choose when truncation is required for integration.

### [SDF Dataset Normalization](3787-sdf-dataset-normalization.md)
Normalizes SDF datasets for consistent scaling. Choose when preparing data for machine learning or standardized pipelines.

## Operating characteristics
- **Memory Efficiency:** High. Stores only active voxels near the surface, making it suitable for large or complex models [S1].
- **Performance:** Fast queries due to sparse tree structure [S1].
- **Use Cases:** Ray-marched rendering, workflows requiring fast implicit queries without storing every voxel [S1].
- **Limitations:** Not suitable for dense volume rendering where every voxel is needed [S1]. Requires closed mesh for signed distances [S1].

## Background
Originated in the MeshLib library, which provides Python and C++ bindings for mesh processing and voxelization [S1]. Developed to address the need for scalable, production-grade SDF generation for large models [S1].

## Sources
[S1] Mesh to SDF (Signed Distance Field) Library for Python & C++, https://meshlib.io/feature/mesh-to-sdf/, fetched 2026-10-30
