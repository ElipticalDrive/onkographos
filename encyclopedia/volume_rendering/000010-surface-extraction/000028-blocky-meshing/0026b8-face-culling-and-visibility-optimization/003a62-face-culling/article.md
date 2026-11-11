# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, which results in excessive geometry and performance overhead. This method sets itself apart from its siblings by performing a direct neighbor check during mesh creation to determine face visibility, rather than relying on coverage masks, vertex packing, or post-generation visibility culling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of occupied voxel positions (typically stored in a hash set or grid) and a list of voxels to process.
*   **Output:** A vertex buffer containing only the exposed faces of the voxels.

**Data Structures:**
The implementation relies on specific structures to track voxel state and face properties [S1, unofficial]:
*   `Cube` struct:
    *   `std::vector<Face> faces`: A vector of size 6, representing the six potential faces of the voxel.
    *   `glm::ivec3 cubeCenter`: The 3D integer coordinates of the voxel's center.
    *   `glm::ivec3 gridPos`: The grid position of the voxel.
    *   `int neighbors`: A counter for adjacent neighbors (used for specific culling logic).
*   `Face` struct:
    *   `std::vector<float> vertices`: The vertex data for the face.
    *   `glm::ivec3 normal`: The normal vector of the face.
    *   `bool isSideFace`: Boolean flag indicating if the face is a side face.
    *   `bool isBottomFace`: Boolean flag indicating if the face is the bottom face.
    *   `bool renderFlag`: A flag determining whether the face should be rendered.

**Algorithm Steps:**
1.  Iterate over each voxel in the scene.
2.  For each voxel, iterate through its six potential faces.
3.  For each face, determine the position of the adjacent voxel in the direction of the face's normal:
    $$ \text{neighborPos} = \text{cubeCenter} + \text{face.normal} $$
4.  Check if a voxel exists at `neighborPos` in the occupied positions set.
5.  Apply culling logic based on the face type:
    *   **Bottom Faces:** Always culled. Set `renderFlag` to `false` regardless of neighbors below [S1, unofficial].
    *   **Side Faces (Non-Bottom):**
        *   Check if a neighbor exists directly adjacent (`neighborPos`).
        *   Check if a neighbor exists directly above the neighbor position (`neighborPos + (0, 1, 0)`).
        *   If either condition is true, set `renderFlag` to `false` [S1, unofficial].
        *   Otherwise, set `renderFlag` to `true` (exposed) [S1, unofficial].
6.  Add faces with `renderFlag` set to `true` to the vertex buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(occupied_positions: set, voxels: list) -> list:
    """
    Generates a mesh by culling faces adjacent to other voxels.
    
    Args:
        occupied_positions: A set of glm::ivec3 representing occupied voxel centers.
        voxels: A list of Cube objects to process.
        
    Returns:
        A list of Face objects that are exposed and should be rendered.
    """
    exposed_faces = []
    
    for cube in voxels:
        for face in cube.faces:
            # Calculate the position of the neighbor in the direction of the face normal
            neighbor_pos = cube.cube_center + face.normal
            
            # Check if a voxel exists at the neighbor position
            neighbor_exists = neighbor_pos in occupied_positions
            
            # Determine render flag based on face type
            if face.is_bottom_face:
                # Bottom faces are always culled [S1, unofficial]
                face.render_flag = False
            else:
                # For side faces, check for direct neighbor or neighbor above
                # Check if neighbor exists directly adjacent
                direct_neighbor = neighbor_exists
                
                # Check if neighbor exists directly above the neighbor position
                # This corresponds to checking the voxel at (neighbor_x, neighbor_y + 1, neighbor_z)
                above_neighbor_pos = neighbor_pos + glm_ivec3(0, 1, 0)
                above_neighbor_exists = above_neighbor_pos in occupied_positions
                
                # Cull if direct neighbor exists OR if neighbor above exists [S1, unofficial]
                if direct_neighbor or above_neighbor_exists:
                    face.render_flag = False
                else:
                    face.render_flag = True
            
            # If the face is exposed, add it to the result
            if face.render_flag:
                exposed_faces.append(face)
                
    return exposed_faces
```

**Parameters and Typical Values:**
*   **Occupied Positions Storage:** Typically a hash set (`std::unordered_set<glm::ivec3>`) for $O(1)$ average lookup time [S1, unofficial].
*   **Face Normal:** Standard unit vectors $(\pm 1, 0, 0), (0, \pm 1, 0), (0, 0, \pm 1)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bottom Face Culling:** The method explicitly culls all bottom faces regardless of whether there is a voxel below. This is a specific design choice in this implementation, likely assuming a ground plane or that bottom faces are not needed for the specific rendering context [S1, unofficial].
*   **Above-Neighbor Check:** The logic for side faces includes a check for a neighbor *above* the adjacent position. This implies that if a voxel is present at $(x, y+1, z)$, the face at $(x, y, z)$ in the direction of $(0, 1, 0)$ might be considered internal or occluded by the voxel above the neighbor. This specific logic needs careful verification against the desired visual result, as it differs from simple adjacency culling [S1, unofficial].
*   **Performance:** The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart by processing entire rows or columns of voxels in parallel using bitwise shifts and masks, avoiding per-voxel loops and random memory access [S2, unofficial]. Choose this method when performance is critical and the voxel data can be efficiently represented as bitmasks, particularly for 1D slices or linearized data structures [S2, unofficial].

**Algorithm:**
1.  Represent each voxel as a bit in a bitmask, where 1 indicates a solid voxel and 0 indicates a transparent or air voxel [S2, unofficial].
2.  To determine visible left faces, shift the bitmask right by 1 position (representing the neighbor to the left), invert the result to find air positions, and perform a bitwise AND with the original mask [S2, unofficial].
    $$ \text{VisibleLeft} = \text{Mask} \ \& \ \sim(\text{Mask} \gg 1) $$
3.  To determine visible right faces, shift the bitmask left by 1 position (representing the neighbor to the right), invert the result to find air positions, and perform a bitwise AND with the original mask [S2, unofficial].
    $$ \text{VisibleRight} = \text{Mask} \ \& \ \sim(\text{Mask} \ll 1) $$
4.  Use the resulting masks to choose which faces to draw or not draw for the mesh generation [S2, unofficial].

**Pitfall:** The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming $O(1)$ lookup for occupied positions. The bitwise variant can be $O(N/word\_size)$ for linearized data.
*   **Memory Complexity:** $O(N)$ to store the voxel data and the resulting mesh.
*   **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
*   **Strengths:** Simple to implement, reduces geometry count significantly (from $6 \times N$ to approximately $2 \times N$ for large solid structures), no runtime overhead during rendering [S1, unofficial].
*   **Limitations:** Requires a full mesh rebuild if the voxel data changes. The specific logic for bottom and above-neighbor culling may not be suitable for all use cases (e.g., floating islands, underground caves) [S1, unofficial].
*   **When to Use:** When generating static or semi-static voxel meshes and reducing draw calls is a priority.
*   **When Not to Use:** When dynamic voxel changes are frequent and mesh rebuilds are expensive, or when bottom faces and complex occlusion are required.

## Background
Face Culling is a standard optimization in voxel terrain generation, widely used in games and simulations to reduce the rendering load. The specific implementation details described here originate from a GitHub repository by cadenmbond [S1, unofficial]. The binary hidden-face culling variant was described in a devlog by Yunasawa for the project "YNL - Vozel" [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
