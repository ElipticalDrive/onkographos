# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It is distinguished from its siblings by its reliance on direct spatial neighbor checks during the mesh creation phase to identify "exposed" faces, rather than using coverage masks, backface normals, or block-level visibility pre-processing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of occupied voxel positions, typically stored in a hash set or dense array for $O(1)$ lookup [S1, unofficial].
*   **Output:** A vertex buffer containing only the faces of voxels that are not adjacent to other solid voxels [S1, unofficial].

**Data Structures:**
The method uses a `Cube` struct to represent individual voxels and a `Face` struct to represent their surfaces [S1, unofficial].
*   `Cube` struct:
    *   `faces`: A vector of 6 `Face` objects, one for each direction ($\pm X, \pm Y, \pm Z$) [S1, unofficial].
    *   `cubeCenter`: The 3D integer coordinates of the voxel's center (`glm::ivec3`) [S1, unofficial].
    *   `gridPos`: The grid position of the voxel (`glm::ivec3`) [S1, unofficial].
    *   `neighbors`: An integer counter for adjacent neighbors (`int`) [S1, unofficial].
*   `Face` struct:
    *   `vertices`: A vector of floats representing the geometric vertices of the face (`std::vector<float>`) [S1, unofficial].
    *   `normal`: The normal vector of the face (`glm::ivec3`) [S1, unofficial].
    *   `isSideFace`: A boolean flag indicating if the face is a side face (`bool`) [S1, unofficial].
    *   `isBottomFace`: A boolean flag indicating if the face is a bottom face (`bool`) [S1, unofficial].
    *   `renderFlag`: A boolean flag determining if the face should be rendered (`bool`) [S1, unofficial].

**Algorithm Steps:**
1.  Iterate through each voxel in the world grid.
2.  For each voxel, iterate through its 6 faces.
3.  For each face, calculate the position of the immediate neighbor in the direction of the face's normal:
    $$ P_{neighbor} = P_{voxel} + \mathbf{n} $$
    where $P_{voxel}$ is the voxel's center position and $\mathbf{n}$ is the face's normal vector [S1, unofficial].
4.  Check if a solid voxel exists at $P_{neighbor}$ using the occupied positions set [S1, unofficial].
5.  Determine the `renderFlag` for the face based on the following logic [S1, unofficial]:
    *   If the face is a bottom face (`isBottomFace` is true): Set `renderFlag` to `false` (always culled) [S1, unofficial].
    *   If the face is a side face (`isBottomFace` is false):
        *   Check if a neighbor exists at $P_{neighbor}$ [S1, unofficial].
        *   Additionally, check if a neighbor exists at $P_{neighbor} + (0, 1, 0)$ (directly above the neighbor position) [S1, unofficial].
        *   If either neighbor exists, set `renderFlag` to `false` [S1, unofficial].
        *   Otherwise, set `renderFlag` to `true` (exposed) [S1, unofficial].
6.  If `renderFlag` is `true`, add the face's vertices to the vertex buffer [S1, unofficial].
7.  (Optional) Count neighbors for metadata by checking specific offsets relative to the voxel center [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(occupied_positions: set[Vector3i]) -> list[Vertex]:
    """
    Generates a mesh by culling faces adjacent to other voxels.
    :param occupied_positions: Set of 3D integer coordinates representing solid voxels.
    :return: List of vertices for the visible faces.
    """
    vertices = []
    
    # Define the 6 face directions and their corresponding normal vectors
    # Order: +X, -X, +Y, -Y, +Z, -Z
    # Note: The specific mapping of 'isBottomFace' and 'isSideFace' 
    # depends on the coordinate system. Based on [S1, unofficial], Y is likely Up.
    face_definitions = [
        {"normal": Vector3i(1, 0, 0), "is_bottom": False},
        {"normal": Vector3i(-1, 0, 0), "is_bottom": False},
        {"normal": Vector3i(0, 1, 0), "is_bottom": False}, # Top face
        {"normal": Vector3i(0, -1, 0), "is_bottom": True},  # Bottom face
        {"normal": Vector3i(0, 0, 1), "is_bottom": False},
        {"normal": Vector3i(0, 0, -1), "is_bottom": False},
    ]

    for voxel_pos in occupied_positions:
        for face_def in face_definitions:
            normal = face_def["normal"]
            is_bottom = face_def["is_bottom"]
            
            # Calculate neighbor position
            neighbor_pos = voxel_pos + normal
            
            # Determine if face is visible
            render = False
            
            if is_bottom:
                # Bottom faces are always culled according to [S1, unofficial]
                render = False
            else:
                # Side faces (including top)
                # Check direct neighbor
                neighbor_exists = neighbor_pos in occupied_positions
                
                # Check neighbor above (0, 1, 0) relative to neighbor position
                # This implies checking if the voxel above the neighbor is solid
                # This specific logic is derived from [S1, unofficial] code snippet
                neighbor_above_pos = neighbor_pos + Vector3i(0, 1, 0)
                neighbor_above_exists = neighbor_above_pos in occupied_positions
                
                if neighbor_exists or neighbor_above_exists:
                    render = False
                else:
                    render = True
            
            if render:
                # Generate vertices for this face
                # Assuming unit cube centered at voxel_pos with side length 1
                # Vertices are generated based on the face normal
                face_vertices = generate_face_vertices(voxel_pos, normal)
                vertices.extend(face_vertices)
                
    return vertices

def generate_face_vertices(center: Vector3i, normal: Vector3i) -> list[Vector3f]:
    """
    Generates the 4 vertices of a face on a unit cube.
    This is a helper function [G] as the exact vertex generation logic 
    is not detailed in the sources but is required for implementation.
    """
    # Define the 8 corners of a unit cube centered at (0,0,0)
    # Offset to center at 'center'
    corners = [
        Vector3f(-0.5, -0.5, -0.5), Vector3f(0.5, -0.5, -0.5),
        Vector3f(0.5, 0.5, -0.5), Vector3f(-0.5, 0.5, -0.5),
        Vector3f(-0.5, -0.5, 0.5), Vector3f(0.5, -0.5, 0.5),
        Vector3f(0.5, 0.5, 0.5), Vector3f(-0.5, 0.5, 0.5)
    ]
    
    # Select the 4 corners that form the face defined by 'normal'
    # This selection logic [G] depends on the specific face direction
    if normal == Vector3i(1, 0, 0): # +X
        selected = [corners[1], corners[5], corners[6], corners[2]]
    elif normal == Vector3i(-1, 0, 0): # -X
        selected = [corners[0], corners[3], corners[7], corners[4]]
    elif normal == Vector3i(0, 1, 0): # +Y
        selected = [corners[6], corners[2], corners[3], corners[7]]
    elif normal == Vector3i(0, -1, 0): # -Y
        selected = [corners[4], corners[0], corners[1], corners[5]]
    elif normal == Vector3i(0, 0, 1): # +Z
        selected = [corners[5], corners[1], corners[2], corners[6]]
    elif normal == Vector3i(0, 0, -1): # -Z
        selected = [corners[4], corners[7], corners[3], corners[0]]
    else:
        return []
        
    # Translate to world position
    return [c + Vector3f(center.x, center.y, center.z) for c in selected]
```

**Parameters:**
*   **Neighbor Check Radius:** 1 voxel (immediate neighbors) [S1, unofficial].
*   **Bottom Face Culling:** Always enabled (hardcoded in the reference implementation) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bottom Face Culling:** The reference implementation always culls bottom faces (`isBottomFace`), regardless of whether there is a voxel below it [S1, unofficial]. This may be intentional for terrain generation (hiding the ground) but is a limitation for general voxel meshing [S1, unofficial].
*   **Side Face Logic:** Side faces are culled if a neighbor exists directly adjacent OR if a neighbor exists directly above the neighbor position [S1, unofficial]. This specific "above neighbor" check is non-standard and may lead to incorrect culling in complex geometries if not carefully understood [S1, unofficial].
*   **Performance:** The naive approach of looping through all voxels and checking neighbors can lead to poor performance due to random memory access [S2, unofficial]. Using efficient data structures like hash sets or bitmasks is recommended [S1, unofficial][S2, unofficial].

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart from standard neighbor checks by processing entire rows or columns of voxels at once using shifts and bitwise AND/OR operations, avoiding per-voxel loops and random memory access [S2, unofficial]. Choose this method when working with 1D or 2D slices of voxels where bitwise operations can be applied efficiently. It is linked to [Face Culling](52-face-culling.md) as a specialized, optimized variant.

## Operating characteristics
*   **Time Complexity:** $O(V)$ where $V$ is the number of voxels, assuming $O(1)$ lookup for neighbor checks [S1, unofficial]. However, the naive implementation with random memory access can be slow [S2, unofficial].
*   **Memory Complexity:** $O(V)$ to store the occupied positions and the resulting vertex buffer [S1, unofficial].
*   **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
*   **Strengths:** Simple to implement; runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].
*   **Limitations:** The reference implementation always culls bottom faces, which may not be desirable for all applications [S1, unofficial]. The specific logic for side faces (checking above neighbors) is non-standard and may require adjustment for general use [S1, unofficial].
*   **When to use:** When generating static voxel meshes and performance during mesh creation is acceptable, but rendering performance is critical.
*   **When not to use:** When dynamic voxel changes are frequent and per-frame culling is needed, or when bottom faces must be visible.

## Background
Face Culling in voxel meshing originated as a basic optimization to reduce the vertex count of voxel terrain. The specific implementation details described here come from a GitHub repository by cadenmbond [S1, unofficial]. The binary variant was described in a devlog by Yunasawa for the project "YNL - Vozel" [S2, unofficial]. These methods are commonly used in voxel-based games and simulations to improve rendering performance by reducing the number of draw calls and vertices processed by the GPU.

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
