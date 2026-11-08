# Voxel Grid to Triangle Mesh Conversion

Also called: Open3D Voxel Grid to Triangle Mesh Conversion

## Overview
This method converts a discrete boolean voxel occupancy grid into a triangular mesh by instantiating a box primitive for every occupied voxel. Unlike smooth surface extraction techniques (e.g., [Marching Cubes](52-marching-cubes.md)) or boundary-aware methods (e.g., [Voxel Occupancy Boundary Detection](1011-voxel-occupancy-boundary-detection.md)), this approach preserves the exact blocky geometry of the input grid without attempting to smooth or merge adjacent faces. It is primarily used when the voxel structure itself is the desired output, such as for animation or when the input data is already voxelized and no underlying scalar field exists.

## How it works
**Inputs:** A voxel grid data structure (e.g., Open3D `VoxelGrid`) containing occupancy information and geometric metadata (voxel size, center coordinates).
**Outputs:** A `TriangleMesh` object containing the triangular facets of the instantiated boxes.

**Data Structures:**
1.  **Voxel Grid:** A sparse or dense representation where each active entry stores:
    *   `voxel_center`: 3D coordinate $(x, y, z)$ of the voxel center.
    *   `voxel_size`: Scalar value $s$ representing the edge length of the cubic voxel.
2.  **TriangleMesh:** A container holding a list of vertices and a list of triangle indices.

**Steps:**
1.  Initialize an empty `TriangleMesh` object.
2.  Retrieve the list of all active voxels from the grid.
3.  For each voxel in the list:
    a.  Determine the voxel's center coordinate $C = (c_x, c_y, c_z)$.
    b.  Determine the voxel's edge length $s$.
    c.  Generate a unit cube (box primitive) with vertices at $(\pm 0.5, \pm 0.5, \pm 0.5)$.
    d.  Scale the unit cube by factor $s$.
    e.  Translate the scaled cube so its center is at $C$.
    f.  Add the resulting vertices and triangle indices to the `TriangleMesh`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_grid_to_mesh(voxel_grid):
    """
    Converts an Open3D VoxelGrid to a TriangleMesh by creating a box for each voxel.
    
    Args:
        voxel_grid: An Open3D VoxelGrid object.
        
    Returns:
        A TriangleMesh object representing the voxel grid.
    """
    # Step 1: Create an empty TriangleMesh object
    mesh = geometry.TriangleMesh()
    
    # Step 2: Extract all voxels from the grid
    # get_voxels() returns a list of Voxel objects
    voxels = voxel_grid.get_voxels()
    
    # Initialize lists to hold mesh data
    vertices = []
    triangles = []
    
    # Define the 8 vertices of a unit cube centered at origin
    # Indices: 0:(-0.5,-0.5,-0.5), 1:(0.5,-0.5,-0.5), 2:(0.5,0.5,-0.5), 3:(-0.5,0.5,-0.5)
    #          4:(-0.5,-0.5,0.5), 5:(0.5,-0.5,0.5), 6:(0.5,0.5,0.5), 7:(-0.5,0.5,0.5)
    unit_cube_vertices = [
        [-0.5, -0.5, -0.5],
        [ 0.5, -0.5, -0.5],
        [ 0.5,  0.5, -0.5],
        [-0.5,  0.5, -0.5],
        [-0.5, -0.5,  0.5],
        [ 0.5, -0.5,  0.5],
        [ 0.5,  0.5,  0.5],
        [-0.5,  0.5,  0.5]
    ]
    
    # Define the 12 triangles (2 per face) for a unit cube
    # Using standard winding order for outward normals
    unit_cube_triangles = [
        [0, 1, 2], [0, 2, 3], # Back face (z=-0.5)
        [4, 6, 5], [4, 7, 6], # Front face (z=0.5)
        [4, 5, 1], [4, 1, 0], # Bottom face (y=-0.5)
        [3, 7, 6], [3, 6, 2], # Top face (y=0.5)
        [3, 4, 0], [3, 7, 4], # Left face (x=-0.5)
        [1, 5, 6], [1, 6, 2]  # Right face (x=0.5)
    ]
    
    # Step 3: Iterate through each voxel
    for voxel in voxels:
        # Get the true size of the created voxels
        voxel_size = voxel.size
        
        # Get the voxel center coordinate
        center = voxel.grid_index * voxel_size + voxel_size / 2.0
        # Note: Open3D's get_voxel_center_coordinate() might return the exact center.
        # If using grid_index, the formula above calculates the center assuming
        # grid_index starts at 0. The source explicitly calls get_voxel_center_coordinate().
        # We will use the explicit method as per source [S1, unofficial].
        center = voxel.get_voxel_center_coordinate()
        
        # Create an Open3D box primitive at each voxel position
        # However, to build a single mesh, we manually transform vertices
        # to avoid creating many separate geometry objects.
        
        # Scale the boxes to the size of the voxels
        # Transform unit cube vertices to world space
        scaled_vertices = []
        for v in unit_cube_vertices:
            # Scale by voxel_size
            sv = [v[0] * voxel_size, v[1] * voxel_size, v[2] * voxel_size]
            # Position at voxel center
            final_v = [sv[0] + center[0], sv[1] + center[1], sv[2] + center[2]]
            scaled_vertices.append(final_v)
            
        # Add the created, scaled, and positioned box to the TriangleMesh object
        # Calculate offset for vertex indices
        vertex_offset = len(vertices)
        
        for tri in unit_cube_triangles:
            triangles.append([
                tri[0] + vertex_offset,
                tri[1] + vertex_offset,
                tri[2] + vertex_offset
            ])
            
        vertices.extend(scaled_vertices)
        
    # Assign data to mesh
    mesh.vertices = geometry.Vector3dVector(vertices)
    mesh.triangles = geometry.IntVector(triangles)
    
    return mesh
```

**Parameters:**
*   **Voxel Size:** Determined by the input voxel grid. Typical values depend on the resolution of the source data.
*   **Box Primitive:** Standard cube with 8 vertices and 12 triangles.

**Edge Cases and Pitfalls:**
*   **Adjacency:** Adjacent occupied voxels will share vertices and edges, but the mesh will contain duplicate geometry (overlapping faces) if not merged. This method does *not* merge coplanar faces.
*   **Precision:** Floating-point precision errors may occur when calculating center coordinates if the voxel grid has a large offset. [G] Ensure the voxel grid's coordinate system is consistent.
*   **Memory:** The number of triangles is $12 \times N$, where $N$ is the number of occupied voxels. This can be significantly larger than a boundary-only mesh.

## Subtopics

### Simple Conversion
Simple Conversion is a variant prioritizing speed over high-fidelity detail preservation, suitable when a mesh is needed ASAP or subtle details are not critical [S2]. It is implemented in tools like MeshInspector.
*   **Workflow:** Upload voxel data (DICOM, RAW, TIFF) -> Apply noise reduction (e.g., 'Change ISO') to isolate ROI -> Select 'Simple Conversion' -> Convert [S2].
*   **Parameters:**
    *   **Preserve Voxel Object:** Optional setting to keep the original voxel object in the scene, resulting in the simple mesh overlaying the original voxel object [S2].
*   **Comparison:** Unlike the Open3D method which constructs a geometric mesh from primitives, Simple Conversion is a high-level tool feature that may involve additional preprocessing steps like noise reduction [S2].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of occupied voxels. Each voxel requires constant-time operations to generate its box geometry.
*   **Memory Complexity:** $O(N)$ for vertices and triangles. Specifically, $8N$ vertices and $12N$ triangles.
*   **Performance:** Very fast for sparse grids. For dense grids, the resulting mesh can be extremely large due to the lack of face merging.
*   **Strengths:** Exact preservation of voxel geometry; simple implementation; no need for scalar field interpolation.
*   **Limitations:** High triangle count; no smoothing; overlapping faces if voxels are adjacent (unless post-processed).
*   **When to use:** When the blocky nature of the voxel grid is desired; for animation of voxel structures; when input is already voxelized and no underlying surface is known.
*   **When not to use:** When a smooth surface is required; when memory efficiency is critical for dense grids.

## Background
This method is often associated with the Open3D library, where it is used to visualize voxelized data as a mesh [S1, unofficial]. The approach is attributed to Florent Poux, Ph.D., as cited in the Open3D documentation [S1, unofficial]. It is a straightforward technique that leverages the geometric simplicity of voxels to create a mesh without complex surface extraction algorithms.

## Sources
[S1] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
[S2] How to convert voxel to mesh with MeshInspector, https://meshinspector.com/knowledge-base/voxel-ct/how-to-convert-voxel-to-mesh-with-meshinspector/, fetched 2026-10-30
