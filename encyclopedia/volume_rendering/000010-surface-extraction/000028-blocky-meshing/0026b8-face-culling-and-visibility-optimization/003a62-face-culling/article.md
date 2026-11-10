# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It sets itself apart from its siblings by operating on a per-voxel neighbor check basis during mesh creation, rather than using coverage masks, block visibility preprocessing, or opacity thresholds.

## How it works
**Inputs and Outputs:**
- Input: A set of occupied voxel positions (typically stored in a hash set or grid) and a list of voxel objects to process.
- Output: A mesh vertex buffer containing only the exposed faces of the voxels.

**Data Structures:**
The method relies on specific data structures to track voxel state and face visibility [S1, unofficial]:
- `Cube` struct:
  - `std::vector<Face> faces`: A vector of 6 `Face` objects, one for each cardinal direction.
  - `glm::ivec3 cubeCenter`: The 3D integer coordinates of the voxel's center.
  - `glm::ivec3 gridPos`: The grid position of the voxel.
  - `int neighbors`: A counter for neighboring voxels (used for specific culling logic).
- `Face` struct:
  - `std::vector<float> vertices`: The vertex data for the face.
  - `glm::ivec3 normal`: The normal vector of the face.
  - `bool isSideFace`: Boolean flag indicating if the face is a side face.
  - `bool isBottomFace`: Boolean flag indicating if the face is a bottom face.
  - `bool renderFlag`: Boolean flag determining if the face should be rendered.

**Algorithm Steps:**
1.  Iterate through each voxel in the scene.
2.  For each voxel, iterate through its 6 faces.
3.  For each face, calculate the position of the neighbor in the direction of the face's normal:
    $$ P_{neighbor} = P_{voxel} + N_{face} $$
    where $P_{voxel}$ is the voxel's center position and $N_{face}$ is the face's normal vector.
4.  Check if a solid voxel exists at $P_{neighbor}$ in the occupied positions set.
5.  Apply culling logic based on face type:
    - If the face is a bottom face (`isBottomFace` is true), set `renderFlag` to `false` (always culled) [S1, unofficial].
    - If the face is a side face (`isBottomFace` is false):
        - Check if a neighbor exists directly adjacent at $P_{neighbor}$.
        - Additionally, check if a neighbor exists directly above the neighbor position ($P_{neighbor} + (0, 1, 0)$). This specific check is part of the "Exposure Culling" variant described in the source [S1, unofficial].
        - If a neighbor exists in either of these positions, set `renderFlag` to `false`.
        - Otherwise, set `renderFlag` to `true` (exposed).
6.  Count neighbors for the voxel by checking specific offset positions (e.g., above offsets) and incrementing the `neighbors` counter [S1, unofficial].
7.  Add the vertices of faces with `renderFlag` set to `true` to the mesh vertex buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(voxels, occupied_positions):
    """
    Generates a mesh by culling hidden faces based on neighbor checks.
    
    Args:
        voxels: List of Cube objects.
        occupied_positions: Set of glm::ivec3 representing occupied voxel centers.
        
    Returns:
        List of vertices for the final mesh.
    """
    mesh_vertices = []
    
    for cube in voxels:
        for face in cube.faces:
            # Calculate neighbor position
            neighbor_pos = cube.cubeCenter + face.normal
            
            # Check if neighbor exists
            neighbor_exists = neighbor_pos in occupied_positions
            
            # Apply culling logic
            if face.isBottomFace:
                # Bottom faces are always culled
                face.renderFlag = False
            else:
                # Side faces logic
                # Check direct neighbor
                direct_neighbor_exists = neighbor_pos in occupied_positions
                
                # Check neighbor above the neighbor position (specific to this implementation)
                neighbor_above_pos = neighbor_pos + glm::ivec3(0, 1, 0)
                neighbor_above_exists = neighbor_above_pos in occupied_positions
                
                # Cull if direct neighbor exists OR neighbor above exists
                if direct_neighbor_exists or neighbor_above_exists:
                    face.renderFlag = False
                else:
                    face.renderFlag = True
            
            # Add vertices if face is exposed
            if face.renderFlag:
                mesh_vertices.extend(face.vertices)
                
    return mesh_vertices
```

**Parameters:**
- `occupied_positions`: The set of occupied voxel positions. Typical implementation uses a hash set for $O(1)$ lookup [S1, unofficial].
- `voxels`: The list of voxels to process.

**Edge Cases and Pitfalls:**
- **Bottom Face Culling:** Bottom faces are always culled regardless of neighbors below, as indicated by the logic `else { face.renderFlag = false; }` for `isBottomFace` [S1, unofficial]. This may lead to missing bottom surfaces if the scene requires them.
- **Side Face Culling Logic:** Side faces are culled if a neighbor exists directly adjacent OR if a neighbor exists directly above the neighbor position [S1, unofficial]. This specific "above neighbor" check is a variation that may not be present in all face culling implementations.
- **Performance:** The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial]. Using a hash set for `occupied_positions` mitigates this.

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart from standard neighbor check culling by operating on bulk bitmasks rather than individual voxel checks, which can be faster for large, contiguous regions. Choose this when performance is critical and the voxel data can be efficiently represented as a bitmask. It requires shifting the bitmask and performing bitwise AND operations to determine visible faces [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V)$ where $V$ is the number of voxels, assuming $O(1)$ lookup for neighbor checks using a hash set. The binary variant can be $O(V/w)$ where $w$ is the word size, due to bitwise operations.
- **Memory Complexity:** $O(V)$ for storing voxel data and neighbor checks.
- **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
- **Strengths:** Significantly reduces vertex count and draw calls by eliminating internal faces. The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].
- **Limitations:** Bottom faces are always culled in the described implementation, which may not be desirable for all scenes. The "above neighbor" check for side faces is a specific variation that may not generalize to all culling needs.
- **When to Use:** When generating static voxel meshes where internal faces are not needed.
- **When Not to Use:** When dynamic voxel changes require frequent mesh updates, or when bottom faces are required.

## Background
Face Culling is a common optimization in voxel-based game development and 3D modeling. The specific implementation described here, including the `Cube` and `Face` structs and the neighbor check logic, is documented in an unofficial GitHub repository [S1, unofficial]. The binary hidden-face culling variant is described in a devlog for a voxel game project [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
