# Disjoint Set Mesh Optimization


## Overview
Disjoint Set Mesh Optimization is a voxel meshing pipeline that combines voxel-based occlusion culling with plane-wise quad merging to produce a compact mesh from a voxel chunk [S1, unofficial]. It solves the problem of excessive face count in naive voxel rendering by removing occluded geometry and merging adjacent coplanar quads into larger primitives [S1, unofficial]. It differs from its parent [Disjoint Set Visible Plane Optimization](4045-disjoint-set-visible-plane-optimization.md) by operating on the full 3D chunk level, adding an initial occlusion culling step before invoking the plane-wise optimizer [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A `Chunk<Voxel>` representing a 3D volume of voxels [S1, unofficial].
- Output: A list of `MeshQuad` objects forming the optimized mesh [S1, unofficial].

**Data Structures:**
- `Chunk<Voxel>`: A 3D grid of voxel data [S1, unofficial].
- `MeshQuad`: A structure representing a quadrilateral face with four vertices (`Vertex0` to `Vertex3`) [S1, unofficial].

**Steps:**
1. Initialize a voxel occlusion optimizer with the input `Chunk<Voxel>` [S1, unofficial].
2. Compute the set of visible faces from the chunk using the occluder, identifying which voxel faces are not occluded by other solid voxels [S1, unofficial].
3. Iterate over each visible plane in the computed visible faces, grouped by axis (X, Y, or Z) [S1, unofficial].
4. For each plane, run a disjoint set visible plane optimizer to merge adjacent quads, using a Disjoint Set Union data structure to efficiently merge contiguous regions of solid voxels with the same ID into single mesh quads [S1, unofficial].
5. Convert the optimized plane data back into a list of `MeshQuad` objects [S1, unofficial].
6. Append the resulting quads to the main mesh's quad list [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_mesh(chunk: Chunk[Voxel]) -> List[MeshQuad]:
    # Step 1 & 2: Compute visible faces using occlusion culling
    visible_faces = compute_visible_faces(chunk)
    
    # Step 3: Group visible faces by plane and axis
    planes = {}
    for face in visible_faces:
        key = (face.axis, face.plane_index)
        planes.setdefault(key, []).append(face)
        
    optimized_quads = []
    
    # Step 4 & 5: Process each plane, merge quads, convert to MeshQuad
    for (axis, plane_index), plane_faces in planes.items():
        merged_quads = merge_quads_on_plane(plane_faces)
        quads = [MeshQuad(v0, v1, v2, v3) for v0, v1, v2, v3 in merged_quads]
        
        # Step 6: Append to result
        optimized_quads.extend(quads)
        
    return optimized_quads

def compute_visible_faces(chunk: Chunk[Voxel]) -> List[VoxelFace]:
    faces = []
    for x, y, z in chunk.voxels:
        if chunk.is_solid(x, y, z):
            if not chunk.is_solid(x + 1, y, z): faces.append(VoxelFace(x, y, z, Axis.X, AxisOrder.Ascending))
            if not chunk.is_solid(x - 1, y, z): faces.append(VoxelFace(x, y, z, Axis.X, AxisOrder.Descending))
            if not chunk.is_solid(x, y + 1, z): faces.append(VoxelFace(x, y, z, Axis.Y, AxisOrder.Ascending))
            if not chunk.is_solid(x, y - 1, z): faces.append(VoxelFace(x, y, z, Axis.Y, AxisOrder.Descending))
            if not chunk.is_solid(x, y, z + 1): faces.append(VoxelFace(x, y, z, Axis.Z, AxisOrder.Ascending))
            if not chunk.is_solid(x, y, z - 1): faces.append(VoxelFace(x, y, z, Axis.Z, AxisOrder.Descending))
    return faces

def merge_quads_on_plane(plane_faces: List[VoxelFace]) -> List[Tuple]:
    # Standard Disjoint Set Union merge for coplanar quads [G]
    # Groups adjacent faces by coordinate, unions sets, and extracts merged quads
    # The sources do not provide the internal DSU implementation details [G]
    return [] 
```

**Parameters:** Not given in the sources.
**Edge Cases and Pitfalls:** Not given in the sources.

## Subtopics

### General Workflow
The general workflow involves building or loading a `Chunk<Voxel>`, using a voxel occlusion optimizer to determine visible faces, using a disjoint set visible plane optimizer to merge faces on each plane, and assembling quads into a single optimized mesh [S1, unofficial]. This workflow is the standard pipeline for this optimization method [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Disjoint Set Mesh Optimization is part of the Voxel Mesh Optimization Library, an unofficial implementation by nicolas-goyon [S1, unofficial]. It builds on the principles of voxel occlusion culling and disjoint set union for efficient mesh optimization [S1, unofficial].

## Sources
[S1] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
