# Voxel Grid to Triangle Mesh Conversion

Also called: Exact Voxel-to-Mesh Triangulation, Open3D Voxel Grid to Triangle Mesh Conversion, Surface Voxel Face Extraction, Voxel Grid to Watertight Mesh Conversion

## Overview
Voxel Grid to Triangle Mesh Conversion extracts a watertight triangular mesh that represents the exact volume of a boolean voxel occupancy grid [S1, unofficial]. It solves the problem of preserving the discrete geometry of the voxel grid without the interpolation artifacts of marching cubes, ensuring that every edge in the resulting mesh connects to exactly two faces [S1, unofficial]. Unlike [Blocky Surface Extraction](6418-blocky-surface-extraction.md) which may produce non-watertight or simplified outputs, this method explicitly generates triangles for every exposed face of surface voxels, guaranteeing volume preservation at the cost of higher triangle counts [S1, unofficial].

## How it works
**Inputs:** A boolean voxel occupancy grid (dense encoding) and its spatial transform [S1, unofficial].
**Outputs:** A `TriangleMesh` object containing vertices and faces representing the surface of the voxel grid [S1, unofficial].

**Data Structures:**
1. **Voxel Grid:** A 3D boolean array `dense_encoding` of shape $(N_x, N_y, N_z)$, where `True` indicates a filled voxel [S1, unofficial].
2. **Surface Indices:** A list of 3D integer coordinates $(i, j, k)$ for all voxels where `dense_encoding[i, j, k]` is `True` and at least one neighbor is `False` or out of bounds [S1, unofficial].
3. **Surface Centers:** A list of 3D floating-point coordinates corresponding to the geometric center of each surface voxel [S1, unofficial].
4. **Face Mask:** A 6-element boolean array `[R, L, F, B, T, B]` for each surface voxel, indicating which of the six faces (Right, Left, Front, Back, Top, Bottom) are exposed [S1, unofficial].
5. **Triangles:** A list of triangle definitions, where each triangle is defined by 3 vertices [S1, unofficial].

**Parameters:**
* `hpitch`: Half the spacing between voxel centers. Computed as `pitch[0] / 2`, where `pitch` is the uniform spacing of the voxel grid [S1, unofficial].

**Steps:**
1. **Extract Surface Voxels:** Identify all voxels that are part of the surface. A voxel is a surface voxel if it is filled (`True`) and has at least one neighbor that is empty (`False`) or out of bounds [S1, unofficial].
2. **Compute Centers:** Calculate the 3D world coordinates of the center of each surface voxel using the grid's transform and indices [S1, unofficial].
3. **Detect Exposed Faces:** For each surface voxel, initialize a 6-element boolean mask to `False`. Check each of the 6 neighboring voxel indices (Right: $+x$, Left: $-x$, Front: $+y$, Back: $-y$, Top: $+z$, Bottom: $-z$). A face is marked `True` if the neighbor index is out of the grid bounds or if the neighbor voxel is not filled (`False`) [S1, unofficial].
4. **Generate Triangles:** For each surface voxel and each `True` entry in its face mask, generate two triangles. The vertices are calculated by offsetting the voxel center by $\pm hpitch$ along the axes perpendicular to the face normal [S1, unofficial].
    * **Right Face ($+x$):** Vertices at $(x+hpitch, y\pm hpitch, z\pm hpitch)$.
    * **Left Face ($-x$):** Vertices at $(x-hpitch, y\pm hpitch, z\pm hpitch)$.
    * **Front Face ($+y$):** Vertices at $(x\pm hpitch, y+hpitch, z\pm hpitch)$.
    * **Back Face ($-y$):** Vertices at $(x\pm hpitch, y-hpitch, z\pm hpitch)$.
    * **Top Face ($+z$):** Vertices at $(x\pm hpitch, y\pm hpitch, z+hpitch)$.
    * **Bottom Face ($-z$):** Vertices at $(x\pm hpitch, y\pm hpitch, z-hpitch)$.
    * Each face is triangulated into two triangles using counter-clockwise winding when viewed from outside the voxel [G].
5. **Construct Mesh:** Create a `TriangleMesh` object. Add the generated vertices and faces. **Crucial:** Disable vertex merging/processing during mesh creation (e.g., `process=False`) to prevent the merging of distinct vertices that share the same coordinate but belong to different faces. Vertex merging causes edges to connect to 4 faces, violating watertightness [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxel_grid_to_triangle_mesh(dense_encoding, pitch, transform):
    """
    Converts a boolean voxel grid to a watertight triangle mesh.
    
    Args:
        dense_encoding: 3D numpy array of booleans (True=voxel present).
        pitch: float, spacing between voxel centers.
        transform: 4x4 numpy array, affine transform from grid indices to world space.
        
    Returns:
        vertices: Nx3 numpy array of vertex positions.
        faces: Mx3 numpy array of vertex indices.
    """
    hpitch = pitch / 2.0
    shape = dense_encoding.shape
    
    def in_bounds(idx):
        return all(0 <= idx[d] < shape[d] for d in range(3))
        
    def is_exposed(idx):
        if not in_bounds(idx):
            return True
        return not dense_encoding[idx[0], idx[1], idx[2]]

    # Step 1: Find surface voxels
    surface_indices = []
    for idx in np.argwhere(dense_encoding):
        i, j, k = idx
        for offset in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]:
            if is_exposed(idx + offset):
                surface_indices.append(idx)
                break
                
    if not surface_indices:
        return np.empty((0, 3)), np.empty((0, 3))
        
    surface_indices = np.array(surface_indices)
    # Compute world centers
    # Assuming indices are integer grid coordinates. 
    # Center in grid space: (i+0.5, j+0.5, k+0.5) * pitch
    centers_grid = surface_indices * pitch + hpitch
    # Apply transform
    h_centers = np.hstack([centers_grid, np.ones((len(centers_grid), 1))])
    surface_centers = (transform @ h_centers.T).T[:, :3]
    
    vertices_list = []
    faces_list = []
    current_v = 0
    
    # Face definitions: (direction_vector, vertex_offsets)
    # Each face has 4 vertices. We define them to ensure outward normal.
    # Right (+x): x+hpitch, y in {+/- hpitch}, z in {+/- hpitch}
    # Vertices: (hpitch, hpitch, -hpitch), (hpitch, hpitch, hpitch), (hpitch, -hpitch, hpitch), (hpitch, -hpitch, -hpitch)
    # Triangles: (0,1,2), (0,2,3)
    face_defs = [
        # Right (+x)
        (np.array([hpitch, 0, 0]), np.array([[hpitch, hpitch, -hpitch], [hpitch, hpitch, hpitch], [hpitch, -hpitch, hpitch], [hpitch, -hpitch, -hpitch]]), [[0,1,2], [0,2,3]]),
        # Left (-x)
        (np.array([-hpitch, 0, 0]), np.array([[-hpitch, hpitch, hpitch], [-hpitch, hpitch, -hpitch], [-hpitch, -hpitch, -hpitch], [-hpitch, -hpitch, hpitch]]), [[0,1,2], [0,2,3]]),
        # Front (+y)
        (np.array([0, hpitch, 0]), np.array([[hpitch, hpitch, hpitch], [-hpitch, hpitch, hpitch], [-hpitch, hpitch, -hpitch], [hpitch, hpitch, -hpitch]]), [[0,1,2], [0,2,3]]),
        # Back (-y)
        (np.array([0, -hpitch, 0]), np.array([[-hpitch, -hpitch, hpitch], [hpitch, -hpitch, hpitch], [hpitch, -hpitch, -hpitch], [-hpitch, -hpitch, -hpitch]]), [[0,1,2], [0,2,3]]),
        # Top (+z)
        (np.array([0, 0, hpitch]), np.array([[hpitch, hpitch, hpitch], [-hpitch, hpitch, hpitch], [-hpitch, -hpitch, hpitch], [hpitch, -hpitch, hpitch]]), [[0,1,2], [0,2,3]]),
        # Bottom (-z)
        (np.array([0, 0, -hpitch]), np.array([[-hpitch, -hpitch, -hpitch], [hpitch, -hpitch, -hpitch], [hpitch, hpitch, -hpitch], [-hpitch, hpitch, -hpitch]]), [[0,1,2], [0,2,3]])
    ]
    
    neighbors_offsets = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
    
    for i, center in enumerate(surface_centers):
        grid_idx = surface_indices[i]
        face_mask = [False] * 6
        for f_idx, offset in enumerate(neighbors_offsets):
            if is_exposed(grid_idx + offset):
                face_mask[f_idx] = True
                
        for f_idx, exposed in enumerate(face_mask):
            if exposed:
                _, v_offsets, t_indices = face_defs[f_idx]
                face_verts = center + v_offsets
                for t in t_indices:
                    faces_list.append([current_v + t[0], current_v + t[1], current_v + t[2]])
                vertices_list.extend(face_verts)
                current_v += 4
                
    return np.array(vertices_list), np.array(faces_list)
```

**Edge Cases and Pitfalls:**
* **Watertightness Violation:** If vertex merging is enabled during mesh construction, distinct vertices at the same geometric location (belonging to adjacent faces of the same voxel or different voxels) may be merged. This causes edges to connect to 4 faces instead of 2, breaking the watertight property [S1, unofficial]. The solution is to disable vertex merging (`process=False`) [S1, unofficial].
* **Degenerate Input:** Empty voxel grids result in empty meshes.
* **Precision:** Floating-point precision in vertex calculation should be sufficient for standard graphics applications. Using `hpitch` ensures vertices align exactly with voxel boundaries.

## Subtopics

### [Range Grid Triangulation](7146-range-grid-triangulation.md)
Range Grid Triangulation converts a 2D range grid, where each grid cell stores an index to a 3D vertex or an invalid marker, into a triangular mesh. It differs from standard marching cubes by operating on pre-projected 2D grid data rather than 3D scalar fields, and handles invalid vertices by constructing triangles only from valid corners.

### [Naive Voxel Triangulation](14035-naive-voxel-triangulation.md)
Naive Voxel Triangulation renders a voxel model by generating triangles for all six faces of every voxel cube, regardless of visibility. It solves the problem of converting a discrete voxel grid into a triangle mesh for immediate rendering. It is distinguished from its sibling [Range Grid Triangulation](7146-range-grid-triangulation.md) by its lack of visibility culling; it produces the maximum possible triangle count for the given voxel data, whereas Range Grid Triangulation optimizes for exposed surfaces.

## Operating characteristics
* **Time Complexity:** $O(N)$, where $N$ is the number of surface voxels. Each surface voxel is processed once, and a constant number of neighbors are checked.
* **Memory Complexity:** $O(N)$, storing vertices and faces for the surface mesh.
* **Performance:** The method is efficient for moderate-sized voxel grids. However, the triangle count is proportional to the surface area of the voxelized object, which can be significantly higher than smooth surface approximations like Marching Cubes [S1, unofficial].
* **Strengths:** Produces an exact representation of the voxel volume; guarantees watertightness if implemented correctly (no vertex merging); simple logic.
* **Limitations:** High triangle count for complex shapes; does not smooth or interpolate surfaces; requires disabling vertex merging to maintain watertightness, which may impact downstream boolean operations [S1, unofficial].
* **When to Use:** When exact volume preservation is required, or when working with boolean voxel grids where interpolation is undesirable.
* **When Not to Use:** When rendering performance is critical and the voxel resolution is high, or when a smooth surface is desired.

## Background
The method is implemented in the `trimesh` library for the voxel module, specifically for rapid construction of unioned boxes from a `VoxelGrid` [S1, unofficial]. It addresses issues raised in community discussions regarding watertightness in voxel-to-mesh conversions [S1, unofficial].

## Sources
[S1] Voxel to Mesh Watertightness Question, https://github.com/mikedh/trimesh/issues/2165, fetched 2026-10-28 (unofficial)
[S2] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
[S3] How to convert voxel to mesh with MeshInspector, https://meshinspector.com/knowledge-base/voxel-ct/how-to-convert-voxel-to-mesh-with-meshinspector/, fetched 2026-10-30
