# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices and triangles sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, which results in excessive geometry and poor performance [S1, unofficial]. It sets itself apart from siblings like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by relying on direct neighbor occupancy checks rather than coverage masks, and from [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) by focusing on inter-voxel adjacency rather than camera-relative backface removal.

## How it works
The method determines face visibility based on the occupancy state of neighboring voxels, as defined in the parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md).

### Inputs and Outputs
*   **Input**: A set of occupied voxel positions (typically stored in a hash set or grid) and the geometry definition for a single voxel (6 faces).
*   **Output**: A list of visible faces (vertices and indices) to be added to the mesh buffer.

### Data Structures
The implementation typically uses the following structures [S1, unofficial]:

1.  **`Cube`**: Represents a voxel instance.
    *   `std::vector<Face> faces`: A container of 6 `Face` objects, one for each direction.
    *   `glm::ivec3 cubeCenter`: The 3D integer coordinates of the voxel's center.
    *   `glm::ivec3 gridPos`: The grid position of the voxel.
    *   `int neighbors`: A counter for adjacent solid voxels (used for optimization or culling logic).
2.  **`Face`**: Represents one side of the voxel.
    *   `std::vector<float> vertices`: The vertex data for the face (typically 4 vertices for a quad).
    *   `glm::ivec3 normal`: The direction vector of the face (e.g., `(1, 0, 0)` for right).
    *   `bool isSideFace`: Boolean flag indicating if the face is a vertical side.
    *   `bool isBottomFace`: Boolean flag indicating if the face is the bottom.
    *   `bool renderFlag`: Boolean flag determining if the face should be rendered.

### Algorithm Steps
The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

1.  Iterate through each voxel in the world grid.
2.  For each voxel, iterate through its 6 faces.
3.  For each face, determine the position of the adjacent voxel in the direction of the face's normal vector $\vec{n}$.
4.  Check if a solid voxel exists at that adjacent position.
5.  Apply visibility logic:
    *   If the neighbor is empty (air/void), the face is visible.
    *   If the neighbor is solid, the face is hidden and must be culled.
6.  Add only visible faces to the vertex buffer.

### Pseudocode
The following pseudocode implements the logic described in [S1, unofficial], rewritten in clean Python-like syntax. It handles the specific pitfall where bottom faces are always culled and side faces check for neighbors above.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(occupied_positions: set[tuple[int, int, int]]) -> list[Face]:
    """
    Generates a mesh by culling hidden faces based on neighbor occupancy.
    
    Args:
        occupied_positions: A set of (x, y, z) tuples representing solid voxels.
        
    Returns:
        A list of Face objects that are visible and should be rendered.
    """
    visible_faces = []
    
    # Define the 6 directions and their corresponding face normals
    # Order: Right, Left, Top, Bottom, Front, Back
    directions = [
        (1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)
    ]
    
    for pos in occupied_positions:
        x, y, z = pos
        
        for i, (dx, dy, dz) in enumerate(directions):
            # Calculate neighbor position
            neighbor_pos = (x + dx, y + dy, z + dz)
            
            # Determine face type based on direction index
            # Assuming indices 0,1 are sides, 2 is top, 3 is bottom, 4,5 are front/back
            is_bottom_face = (i == 3)
            is_side_face = i in [0, 1, 4, 5]
            
            # Initialize render flag
            render = True
            
            if is_bottom_face:
                # Pitfall: Bottom faces are always culled regardless of neighbors below
                render = False
            elif is_side_face:
                # Check for direct neighbor
                neighbor_exists = neighbor_pos in occupied_positions
                
                # Check for neighbor directly above the neighbor position
                # This handles cases where a voxel above might occlude the side face
                # depending on the specific implementation logic in [S1, unofficial]
                above_neighbor_pos = (x + dx, y + 1, z + dz)
                neighbor_above_exists = above_neighbor_pos in occupied_positions
                
                if neighbor_above_exists:
                    render = False
                else:
                    render = not neighbor_exists
            else:
                # Top face logic (not explicitly detailed in [S1, unofficial] code snippet
                # but implied by standard neighbor check)
                neighbor_exists = neighbor_pos in occupied_positions
                render = not neighbor_exists
            
            if render:
                # Create face geometry
                face = create_face_geometry(pos, directions[i])
                visible_faces.append(face)
                
    return visible_faces

def create_face_geometry(center: tuple[int, int, int], normal: tuple[int, int, int]) -> Face:
    """
    Creates the vertex data for a single face of a voxel.
    Note: Exact vertex layout depends on the renderer's coordinate system.
    This is a placeholder for the geometry generation step.
    """
    # Implementation details for generating 4 vertices for a quad
    # are not provided in the sources. [G]
    return Face(vertices=[], normal=normal)
```

### Parameters and Edge Cases
*   **Bottom Face Culling**: In the specific implementation cited in [S1, unofficial], bottom faces are always culled (`renderFlag = false`) regardless of whether there is a neighbor below. This is a specific pitfall/feature of that implementation [S1, unofficial].
*   **Side Face Occlusion**: Side faces are culled not only if a direct neighbor exists, but also if a neighbor exists directly above the neighbor position (`neighborPos + (0, 1, 0)`). This suggests a specific handling for overhangs or specific visual styles [S1, unofficial].
*   **Performance**: The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart from standard neighbor checks by processing entire rows or columns of voxels at once using bitwise shifts and AND operations, avoiding random memory access [S2, unofficial]. Choose this method when performance is critical and the voxel data can be efficiently represented as bitmasks [S2, unofficial]. It is a variant of Face Culling that optimizes the neighbor check step [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(V)$, where $V$ is the number of voxels. Each voxel is processed once, and for each voxel, 6 neighbor checks are performed.
*   **Memory Complexity**: $O(V)$ to store the occupied positions and the resulting mesh.
*   **Measured Performance**: In the cited implementation, 4096 cubes were rendered at ~60 FPS [S1, unofficial].
*   **Strengths**: Significantly reduces the number of vertices and triangles sent to the GPU, improving rendering performance. The culling check runs once during mesh creation, so there is no runtime overhead during rendering [S1, unofficial].
*   **Limitations**: The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial]. Binary Hidden-Face Culling mitigates this by using bitwise operations [S2, unofficial].
*   **When to Use**: When generating static or semi-static voxel meshes where the geometry does not change frequently.
*   **When Not to Use**: When the voxel world changes frequently and mesh regeneration is too costly.

## Background
Face Culling is a fundamental optimization in voxel rendering, originating from the need to efficiently render large voxel worlds like those in Minecraft. The specific implementation details described in [S1, unofficial] come from a GitHub repository by cadenmbond, while the Binary Hidden-Face Culling variant is described in a devlog by Yunasawa for the project "YNL - Vozel" [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
