# Voxel Mesh Synthesis


## Overview
Voxel Mesh Synthesis is a post-processing step that converts validated leaf voxels from an octree into standard 3D cube meshes for export [S1, unofficial]. It differs from its parent, Greedy Meshing, by generating individual cube geometry rather than merging adjacent faces into larger quads or blocks [S1, unofficial]. It serves as a bridge between volumetric data structures and standard polygonal formats like Wavefront .obj [S1, unofficial].

## How it works
**Inputs:** A set of validated leaf voxels from an octree, each defined by its spatial bounds (min and max coordinates).
**Outputs:** A standard Wavefront .obj file containing triangle meshes.

**Data Structures:**
The method generates geometry for a single cube. The cube is defined by 8 unique corner vertices.

**Steps:**
1.  **Vertex Generation:** For a cube defined by minimum coordinates $(x_{min}, y_{min}, z_{min})$ and maximum coordinates $(x_{max}, y_{max}, z_{max})$, generate 8 vertices. Each vertex is a 3D vector $(x, y, z)$ corresponding to one of the 8 combinations of min/max for each axis.
2.  **Face Construction:** Construct 12 triangular faces, representing 2 triangles per cube face. There are 6 faces on a cube.
3.  **Normal Orientation:** Ensure that the winding order of the vertices for each triangle results in outward-pointing surface normals. This is typically achieved by using counter-clockwise (CCW) winding order when viewed from the outside.
4.  **Export:** Output the vertices and indices in Wavefront .obj format, which is compatible with external 3D software [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_cube_mesh(min_coord, max_coord):
    """
    Generates a mesh for a single voxel cube.
    
    Args:
        min_coord: Tuple (x, y, z) of the cube's minimum corner.
        max_coord: Tuple (x, y, z) of the cube's maximum corner.
        
    Returns:
        vertices: List of 8 vertex tuples.
        indices: List of 36 integer indices (12 triangles * 3 vertices).
    """
    x0, y0, z0 = min_coord
    x1, y1, z1 = max_coord
    
    # Step 1: Generate 8 unique corner vertices
    # Vertices are ordered to facilitate consistent face indexing
    # 0: (x0, y0, z0)
    # 1: (x1, y0, z0)
    # 2: (x1, y1, z0)
    # 3: (x0, y1, z0)
    # 4: (x0, y0, z1)
    # 5: (x1, y0, z1)
    # 6: (x1, y1, z1)
    # 7: (x0, y1, z1)
    vertices = [
        (x0, y0, z0), (x1, y0, z0), (x1, y1, z0), (x0, y1, z0),
        (x0, y0, z1), (x1, y0, z1), (x1, y1, z1), (x0, y1, z1)
    ]
    
    # Step 2 & 3: Construct 12 triangles (2 per face) with CCW winding
    # Face indices are defined to ensure outward normals
    # Front face (z0): 0, 1, 2 and 0, 2, 3
    # Back face (z1): 4, 6, 5 and 4, 7, 6
    # Right face (x1): 1, 5, 6 and 1, 6, 2
    # Left face (x0): 0, 3, 7 and 0, 7, 4
    # Top face (y1): 3, 2, 6 and 3, 6, 7
    # Bottom face (y0): 0, 4, 5 and 0, 5, 1
    
    indices = [
        # Front (z0)
        0, 1, 2, 0, 2, 3,
        # Back (z1)
        4, 6, 5, 4, 7, 6,
        # Right (x1)
        1, 5, 6, 1, 6, 2,
        # Left (x0)
        0, 3, 7, 0, 7, 4,
        # Top (y1)
        3, 2, 6, 3, 6, 7,
        # Bottom (y0)
        0, 4, 5, 0, 5, 1
    ]
    
    return vertices, indices
```

**Edge Cases:**
*   **Degenerate Input:** If `min_coord` equals `max_coord`, the cube has zero volume. The resulting mesh will have coincident vertices and zero-area faces. [G] Implementers should check for this condition and skip mesh generation or handle it as a point/invalid voxel.
*   **Precision:** Floating-point precision issues may arise if voxel coordinates are very large or very small. [G] Use double precision if necessary to maintain vertex accuracy.

## Subtopics

### Hybrid Scene Static Merging
Optimizes static voxel models by merging subtrees without animated transforms into a single mesh, with inner face culling [S2, unofficial]. It rasterizes these subtrees into a single mesh, culling inner faces between merged components [S2, unofficial]. If the volume of the merged mesh exceeds a specified limit, it falls back to keeping the meshes separate [S2, unofficial]. It is controlled by a boolean setting `mergeStatic` in `VoxelImportSettings` [S2, unofficial]. The volume limit threshold is not specified in the sources [S2, unofficial].

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Strengths:** Produces clean, standard Wavefront .obj data compatible with external 3D software [S1, unofficial]. Simple to implement.
**Limitations:** Does not reduce vertex count by merging adjacent faces, leading to higher polygon counts compared to greedy meshing methods.
**When to use:** When exporting voxel data to standard 3D modeling or rendering pipelines that require standard cube geometry.
**When not to use:** When performance or memory usage is critical and the scene contains large contiguous volumes of voxels.

## Background
Voxel Mesh Synthesis is described in the context of a 3D object surface voxelizer project [S1, unofficial]. It is also referenced in the context of the AdaEngine's voxel graphics system, specifically regarding hybrid scene optimization and static merging [S2, unofficial].

## Sources
[S1] Joji0/3D-Object-Surface-Voxelizer, https://github.com/Joji0/3D-Object-Surface-Voxelizer, fetched 2026-10-27 (unofficial)
[S2] Voxel graphics: MagicaVoxel import, animation, dynamic world and physics (ADR-0025), https://github.com/AdaEngine/Ada/pull/136, fetched 2026-10-26 (unofficial)
