# Disjoint Set Mesh Optimization


## Overview
Disjoint Set Mesh Optimization is a voxel meshing pipeline that combines voxel-based occlusion culling with plane-wise quad merging to produce a compact triangle mesh from a voxel chunk. It solves the problem of excessive face count in naive voxel rendering by removing occluded geometry and merging adjacent coplanar quads into larger primitives. It differs from its parent [Disjoint Set Visible Plane Optimization](4045-disjoint-set-visible-plane-optimization.md) by operating on the full 3D chunk level, adding an initial occlusion culling step before invoking the plane-wise optimizer.

## How it works
**Inputs and Outputs:**
- Input: A `Chunk<Voxel>` representing a 3D volume of voxels.
- Output: A list of `MeshQuad` objects forming the optimized mesh.

**Data Structures:**
- `Chunk<Voxel>`: A 3D grid of voxel data.
- `VoxelOcclusionOptimizer`: An internal component that determines which voxel faces are visible.
- `DisjointSetVisiblePlaneOptimizer`: An internal component that merges quads on a single 2D plane using a Disjoint Set Union (DSU) structure.
- `MeshQuad`: A structure representing a quadrilateral face with four vertices (`Vertex0` to `Vertex3`).

**Steps:**
1. Initialize a `VoxelOcclusionOptimizer` with the input `Chunk<Voxel>`.
2. Compute the set of visible faces from the chunk using the occluder. This identifies which voxel faces are not occluded by other solid voxels.
3. Iterate over each visible plane in the computed visible faces, grouped by axis (X, Y, or Z).
4. For each plane, run the `DisjointSetVisiblePlaneOptimizer` to merge adjacent quads. This step uses a Disjoint Set Union data structure to efficiently merge contiguous regions of solid voxels with the same ID into single mesh quads.
5. Convert the optimized plane data back into a list of `MeshQuad` objects.
6. Append the resulting quads to the main mesh's quad list.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_mesh(chunk: Chunk[Voxel]) -> List[MeshQuad]:
    # Step 1: Initialize occlusion optimizer
    occlusion_optimizer = VoxelOcclusionOptimizer(chunk)
    
    # Step 2: Compute visible faces
    visible_faces = occlusion_optimizer.compute_visible_faces()
    
    # Step 3: Group visible faces by plane and axis
    # visible_faces is a collection of planes, each associated with an axis
    planes = group_by_axis(visible_faces)
    
    optimized_quads = []
    
    # Step 4: Process each plane
    for axis, plane_data in planes.items():
        # Run the plane optimizer to merge quads
        merged_plane_data = DisjointSetVisiblePlaneOptimizer(plane_data)
        
        # Step 5: Convert to MeshQuad objects
        quads = convert_to_mesh_quads(merged_plane_data)
        
        # Step 6: Append to result
        optimized_quads.extend(quads)
        
    return optimized_quads
```

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- Empty chunks or chunks with no solid voxels should return an empty list of quads.
- Precision issues in vertex coordinates may prevent merging of adjacent quads if they are not exactly coplanar.
- Degenerate input (e.g., single voxel) should still produce valid quads.

## Subtopics

### General Workflow
The general workflow involves building or loading a `Chunk<Voxel>`, using `VoxelOcclusionOptimizer` to determine visible faces, using `DisjointSetVisiblePlaneOptimizer` to merge faces on each plane, and assembling quads into a single optimized mesh. This workflow is the standard pipeline for this optimization method.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Reduces mesh complexity by culling occluded faces and merging adjacent quads.
- **Limitations:** Performance depends on the efficiency of the occlusion culling and plane-wise merging algorithms.
- **When to Use:** Suitable for voxel-based rendering where mesh complexity is a concern.
- **When Not to Use:** Not suitable for real-time applications with strict performance constraints if the optimization overhead is too high.

## Background
Disjoint Set Mesh Optimization is part of the Voxel Mesh Optimization Library, an unofficial implementation by nicolas-goyon. It builds on the principles of voxel occlusion culling and disjoint set union for efficient mesh optimization.

## Sources
[S1] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
