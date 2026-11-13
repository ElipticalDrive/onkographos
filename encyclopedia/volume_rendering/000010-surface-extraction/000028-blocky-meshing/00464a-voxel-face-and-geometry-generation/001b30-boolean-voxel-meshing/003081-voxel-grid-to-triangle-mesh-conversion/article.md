# Voxel Grid to Triangle Mesh Conversion

Also called: Exact Voxel-to-Mesh Triangulation, Open3D Voxel Grid to Triangle Mesh Conversion, Surface Voxel Face Extraction, Voxel Grid to Watertight Mesh Conversion

## Overview
This method converts a boolean voxel occupancy grid into a watertight triangular mesh that preserves the exact volume of the original grid by generating triangles for every exposed face of surface voxels [S1, unofficial]. It solves the problem of incorrect edge connectivity where edges might connect to 4 faces instead of exactly 2, which violates standard definitions of watertightness [S1, unofficial]. Unlike marching cubes or smooth surface extraction methods that interpolate boundaries, this approach explicitly generates cube faces per surface voxel, preserving the exact voxel geometry at the cost of a higher triangle count [S1, unofficial].

## How it works
**Inputs:** A boolean voxel grid (encoding) with known spatial transform and pitch.
**Outputs:** A `TriangleMesh` object containing vertices and faces representing the surface of the voxel grid.

**Data Structures:**
1.  **Voxel Grid Encoding:** A dense boolean array or sparse index list representing occupied voxels.
2.  **Surface Indices:** A list of 3D integer coordinates $(i, j, k)$ for voxels that are occupied but have at least one unoccupied neighbor.
3.  **Surface Centers:** A list of 3D floating-point coordinates corresponding to the center of each surface voxel.
4.  **Face Mask:** A 6-element boolean array `[Right, Left, Front, Back, Top, Bottom]` for each surface voxel.
5.  **Triangles:** A list of triplets of vertex coordinates.

**Parameters:**
*   `hpitch`: Half of the voxel grid pitch (spacing between voxel centers). Computed as `pitch[0] / 2` [S1, unofficial].

**Steps:**
1.  **Extract Surface Voxels:** Identify surface voxels from the original grid. A surface voxel is an occupied voxel that has at least one neighbor that is either out of bounds or unoccupied. This can be done using a morphology operation to create a "hollow" voxel grid containing only surface voxels [S1, unofficial].
2.  **Compute Centers:** Calculate the 3D world coordinates of each surface voxel center from its sparse indices using the grid's transform and pitch [S1, unofficial].
3.  **Initialize Mesh:** Create an empty `TriangleMesh` object [S2, unofficial].
4.  **Detect Faces:** For each surface voxel:
    a. Initialize a 6-element boolean mask to `False`.
    b. Check each of the 6 neighboring voxel indices (Right: $+x$, Left: $-x$, Front: $+y$, Back: $-y$, Top: $+z$, Bottom: $-z$).
    c. A face is marked `True` if the neighbor index is out of bounds or if the neighbor voxel is not present in the original dense encoding [S1, unofficial].
5.  **Generate Triangles:** For each `True` entry in the face mask, generate two triangles by offsetting the voxel center by `hpitch` along the appropriate axes to form the four corners of the face, then triangulating into two triangles [S1, unofficial].
6.  **Add to Mesh:** Add the generated triangles to the `TriangleMesh` object [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxel_grid_to_triangle_mesh(voxel_grid):
    """
    Converts a voxel grid to a watertight triangle mesh.
    
    Args:
        voxel_grid: An object containing:
            - encoding: A dense boolean numpy array (True for occupied).
            - pitch: A 3D vector (x, y, z) representing the spacing between voxel centers.
            - transform: A 4x4 transformation matrix for the grid.
            
    Returns:
        vertices: numpy array of shape (N, 3)
        faces: numpy array of shape (M, 3)
    """
    # 1. Extract surface voxels
    # Create a hollow voxel grid containing only surface voxels
    # This assumes a morphology.surface function exists that returns a boolean grid
    # of voxels that are occupied but have at least one empty neighbor.
    hollow_encoding = morphology_surface(voxel_grid.encoding)
    
    # Get indices of surface voxels
    surface_indices = np.argwhere(hollow_encoding)
    
    # 2. Compute centers
    # Convert sparse indices to world coordinates
    # Assuming indices_to_points handles the transform and pitch scaling
    surface_centers = indices_to_points(surface_indices, voxel_grid.transform, voxel_grid.pitch)
    
    # 3. Initialize data structures
    triangles = []
    hpitch = voxel_grid.pitch[0] / 2.0
    
    # Helper functions
    def index_in_bounds(index):
        """Check if index is within the bounds of the dense encoding."""
        return (np.all(index >= 0) and 
                np.all(index < voxel_grid.encoding.shape))
    
    def actual_face(index):
        """Check if a neighbor is out of bounds or unoccupied."""
        if not index_in_bounds(index):
            return True
        return not voxel_grid.encoding[tuple(index)]
    
    # 4. Detect faces and generate triangles
    for i in range(len(surface_indices)):
        base_index = surface_indices[i]
        base_center = surface_centers[i]
        
        # Initialize face mask: [Right, Left, Front, Back, Top, Bottom]
        face_mask = [False] * 6
        
        # Right (+x)
        target_index = base_index + np.array([1, 0, 0])
        if actual_face(target_index):
            face_mask[0] = True
            
        # Left (-x)
        target_index = base_index + np.array([-1, 0, 0])
        if actual_face(target_index):
            face_mask[1] = True
            
        # Front (+y)
        target_index = base_index + np.array([0, 1, 0])
        if actual_face(target_index):
            face_mask[2] = True
            
        # Back (-y)
        target_index = base_index + np.array([0, -1, 0])
        if actual_face(target_index):
            face_mask[3] = True
            
        # Top (+z)
        target_index = base_index + np.array([0, 0, 1])
        if actual_face(target_index):
            face_mask[4] = True
            
        # Bottom (-z)
        target_index = base_index + np.array([0, 0, -1])
        if actual_face(target_index):
            face_mask[5] = True
            
        # 5. Generate triangles for exposed faces
        # Right face (x + hpitch)
        if face_mask[0]:
            v0 = base_center + np.array([hpitch, hpitch, -hpitch])
            v1 = base_center + np.array([hpitch, hpitch, hpitch])
            v2 = base_center + np.array([hpitch, -hpitch, hpitch])
            v3 = base_center + np.array([hpitch, -hpitch, -hpitch])
            triangles.append([v0, v1, v2])
            triangles.append([v0, v2, v3])
            
        # Left face (x - hpitch)
        if face_mask[1]:
            v0 = base_center + np.array([-hpitch, hpitch, hpitch])
            v1 = base_center + np.array([-hpitch, hpitch, -hpitch])
            v2 = base_center + np.array([-hpitch, -hpitch, -hpitch])
            v3 = base_center + np.array([-hpitch, -hpitch, hpitch])
            triangles.append([v0, v1, v2])
            triangles.append([v0, v2, v3])
            
        # Front face (y + hpitch)
        if face_mask[2]:
            v0 = base_center + np.array([-hpitch, hpitch, hpitch])
            v1 = base_center + np.array([hpitch, hpitch, hpitch])
            v2 = base_center + np.array([hpitch, hpitch, -hpitch])
            v3 = base_center + np.array([-hpitch, hpitch, -hpitch])
            triangles.append([v0, v1, v2])
            triangles.append([v0, v2, v3])
            
        # Back face (y - hpitch)
        if face_mask[3]:
            v0 = base_center + np.array([hpitch, -hpitch, -hpitch])
            v1 = base_center + np.array([-hpitch, -hpitch, -hpitch])
            v2 = base_center + np.array([-hpitch, -hpitch, hpitch])
            v3 = base_center + np.array([hpitch, -hpitch, hpitch])
            triangles.append([v0, v1, v2])
            triangles.append([v0, v2, v3])
            
        # Top face (z + hpitch)
        if face_mask[4]:
            v0 = base_center + np.array([-hpitch, hpitch, hpitch])
            v1 = base_center + np.array([hpitch, hpitch, hpitch])
            v2 = base_center + np.array([hpitch, -hpitch, hpitch])
            v3 = base_center + np.array([-hpitch, -hpitch, hpitch])
            triangles.append([v0, v1, v2])
            triangles.append([v0, v2, v3])
            
        # Bottom face (z - hpitch)
        if face_mask[5]:
            v0 = base_center + np.array([hpitch, -hpitch, -hpitch])
            v1 = base_center + np.array([-hpitch, -hpitch, -hpitch])
            v2 = base_center + np.array([-hpitch, hpitch, -hpitch])
            v3 = base_center + np.array([hpitch, hpitch, -hpitch])
            triangles.append([v0, v1, v2])
            triangles.append([v0, v2, v3])
            
    # Convert list of triangles to numpy arrays
    if len(triangles) == 0:
        return np.empty((0, 3)), np.empty((0, 3))
        
    vertices = np.vstack(triangles).reshape(-1, 3)
    # Create face indices
    num_verts = len(vertices)
    faces = np.arange(num_verts).reshape(-1, 3)
    
    return vertices, faces

def morphology_surface(encoding):
    """
    [G] Placeholder for morphology operation.
    Returns a boolean grid where True indicates a surface voxel.
    A surface voxel is occupied and has at least one unoccupied neighbor.
    """
    # This is a standard morphological operation:
    # surface = occupied AND NOT (dilated(occupied) == occupied)
    # Or more directly: surface = occupied AND (occupied XOR dilated(occupied))
    # Implementation depends on the specific library (e.g., scipy.ndimage)
    # For the purpose of this article, we assume this function exists.
    pass

def indices_to_points(indices, transform, pitch):
    """
    [G] Placeholder for coordinate conversion.
    Converts voxel indices to world coordinates.
    """
    # world_coords = indices * pitch + offset
    # Then apply transform
    pass
```

**Edge Cases and Pitfalls:**
*   **Watertightness Violation:** Vertex merging during mesh creation can cause edges to connect to 4 faces, violating watertightness (exactly 2 faces per edge) [S1, unofficial]. To prevent this, disable vertex merging/processing upon creation (e.g., `process=False` in Trimesh) [S1, unofficial]. Note that this may affect downstream boolean operations if processing is applied internally later [S1, unofficial].
*   **Empty Grid:** If the input voxel grid is empty, the output mesh should be empty.
*   **Fully Occupied Grid:** If the voxel grid is fully occupied (no surface voxels), the output mesh should be empty.

## Subtopics

### [Range Grid Triangulation](7146-range-grid-triangulation.md)
Range Grid Triangulation converts a 2D range grid, where each grid cell stores an index to a 3D vertex or an invalid marker, into a triangular mesh. It differs from standard marching cubes by operating on pre-projected 2D grid data rather than 3D scalar fields, and handles invalid vertices by constructing triangles only from valid corners.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of surface voxels. Each surface voxel is processed once, and the number of neighbors checked is constant (6).
*   **Memory Complexity:** $O(N)$, where $N$ is the number of surface voxels. The output mesh size is proportional to the surface area of the voxel grid.
*   **Performance:** The method is efficient for dense voxel grids with large surface areas. However, the triangle count can be significantly higher than smoothed surface extraction methods like Marching Cubes, which may impact rendering performance for very large models [S1, unofficial].
*   **Strengths:** Preserves exact voxel volume and geometry; produces watertight meshes when vertex merging is disabled; simple to implement.
*   **Limitations:** High triangle count; does not smooth the surface; sensitive to vertex merging issues if not handled correctly [S1, unofficial].
*   **When to Use:** When exact volume preservation is required, or when working with boolean voxel meshes where smooth interpolation is not desired [S1, unofficial].
*   **When Not to Use:** When rendering performance is critical and the voxel grid is very large, or when a smooth surface is desired.

## Background
This method is used in the `trimesh` library for the voxel module, specifically for rapid construction of unioned boxes from a `VoxelGrid` [S1, unofficial]. It addresses the specific issue of watertightness in voxel-to-mesh conversion, which is a common problem in 3D printing and geometric modeling where exact volume representation is crucial [S1, unofficial]. The approach is based on the principle of generating triangles for every exposed face of surface voxels, ensuring that the resulting mesh is a precise representation of the voxel grid's surface.

## Sources
[S1] Voxel to Mesh Watertightness Question, https://github.com/mikedh/trimesh/issues/2165, fetched 2026-10-28 (unofficial)
[S2] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
[S3] How to convert voxel to mesh with MeshInspector, https://meshinspector.com/knowledge-base/voxel-ct/how-to-convert-voxel-to-mesh-with-meshinspector/, fetched 2026-10-30
