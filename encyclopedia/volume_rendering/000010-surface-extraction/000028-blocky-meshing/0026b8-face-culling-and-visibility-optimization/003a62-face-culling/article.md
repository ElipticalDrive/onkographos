# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices and triangles sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It sets itself apart from its siblings by operating on a per-voxel neighbor check basis during mesh creation, rather than using coverage masks, block visibility, or post-generation projection [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of occupied voxel positions (typically stored in a hash set or dense array) and a list of voxel blocks to process.
- Output: A vertex buffer containing only the exposed faces of the voxels.

**Data Structures:**
The implementation requires a spatial lookup structure to efficiently query the state of neighboring voxels [S1, unofficial]. A common layout involves a `Cube` struct and a `Face` struct [S1, unofficial]:
- `Cube`:
  - `faces`: `std::vector<Face>` of size 6, one for each direction.
  - `cubeCenter`: `glm::ivec3` representing the center position of the cube.
  - `gridPos`: `glm::ivec3` representing the grid position.
  - `neighbors`: `int` counter for adjacent occupied voxels.
- `Face`:
  - `vertices`: `std::vector<float>` containing the vertex data for the face.
  - `normal`: `glm::ivec3` indicating the face's outward normal direction.
  - `isSideFace`: `bool` indicating if the face is a side face (not top/bottom).
  - `isBottomFace`: `bool` indicating if the face is the bottom face.
  - `renderFlag`: `bool` determining if the face should be rendered.

**Steps:**
1. For each voxel at position $(x, y, z)$, iterate through its 6 faces.
2. For each face, determine the position of the adjacent neighbor voxel in the direction of the face's normal: $P_{neighbor} = P_{voxel} + \vec{n}$.
3. Check if a voxel exists at $P_{neighbor}$ using the spatial lookup structure.
4. Apply visibility logic to set the `renderFlag` for the face:
   - If the face is a bottom face, set `renderFlag` to `false` (always culled) [S1, unofficial].
   - If the face is a side face:
     - Check if a neighbor exists directly adjacent ($P_{neighbor}$).
     - Check if a neighbor exists directly above the neighbor position ($P_{neighbor} + (0, 1, 0)$).
     - If either condition is true, set `renderFlag` to `false` [S1, unofficial].
     - Otherwise, set `renderFlag` to `true` if no neighbor exists, or `false` if a neighbor exists [S1, unofficial].
5. If `renderFlag` is `true`, add the face's vertices to the mesh buffer.
6. The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cull_faces(voxel_centers: set, faces: list) -> list:
    """
    Cull faces of voxels based on neighbor occupancy.
    
    Args:
        voxel_centers: A set of (x, y, z) tuples representing occupied voxel positions.
        faces: A list of Face objects, each with attributes:
               - center: (x, y, z) tuple
               - normal: (nx, ny, nz) tuple
               - is_bottom: bool
               - is_side: bool
               - render_flag: bool (initially True)
    
    Returns:
        A list of Face objects with updated render_flags.
    """
    for face in faces:
        # Calculate neighbor position
        neighbor_pos = (
            face.center[0] + face.normal[0],
            face.center[1] + face.normal[1],
            face.center[2] + face.normal[2]
        )
        
        # Check if neighbor exists
        neighbor_exists = neighbor_pos in voxel_centers
        
        if face.is_bottom:
            # Bottom faces are always culled
            face.render_flag = False
        elif face.is_side:
            # Side faces are culled if:
            # 1. A neighbor exists directly adjacent
            # 2. A neighbor exists directly above the neighbor position
            if neighbor_exists:
                face.render_flag = False
            else:
                # Check for neighbor above the current neighbor
                neighbor_above = (
                    neighbor_pos[0],
                    neighbor_pos[1] + 1,
                    neighbor_pos[2]
                )
                if neighbor_above in voxel_centers:
                    face.render_flag = False
                else:
                    face.render_flag = True
        else:
            # Top faces or other faces not explicitly handled in the source logic
            # Assuming standard behavior: visible if no neighbor
            face.render_flag = not neighbor_exists
            
    return faces
```

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- Bottom faces are always culled regardless of neighbors below, as indicated by the logic `else { face.renderFlag = false; }` for `isBottomFace` [S1, unofficial].
- Side faces are culled if a neighbor exists directly adjacent OR if a neighbor exists directly above the neighbor position (checking `neighborPos + glm::ivec3(0, 1, 0)`) [S1, unofficial].
- The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial].

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart from standard neighbor checks by operating on bulk bitmasks rather than individual voxel lookups, offering better performance for large datasets [S2, unofficial]. Choose this when performance is critical and voxels can be represented as a linear bitmask. To determine visible left faces, shift the bitmask right by 1 position (representing the neighbor to the left), invert the result to find air positions, and perform a bitwise AND with the original mask. To determine visible right faces, shift the bitmask left by 1 position (representing the neighbor to the right), invert the result to find air positions, and perform a bitwise AND with the original mask [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. The original approach of looping through neighbors has poor performance due to random memory access [S2, unofficial]. Binary culling offers bulk operations.
- **Memory Complexity:** Not given in the sources. Requires a spatial lookup structure (e.g., hash set) [S1, unofficial].
- **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
- **Scaling:** Not given in the sources.
- **Strengths:** Reduces vertex count significantly by removing hidden faces. Culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].
- **Limitations:** Bottom faces are always culled, which may not be desired in all scenarios [S1, unofficial]. Side face culling logic includes an additional check for neighbors above, which may be specific to certain terrain generation styles [S1, unofficial].
- **When to use:** When generating voxel meshes and wanting to reduce GPU load by removing hidden faces.
- **When not to use:** When bottom faces need to be visible or when the culling logic's specific neighbor checks do not match the desired visual result.

## Background
Face Culling is a common optimization in voxel terrain generation. The specific implementation described in [S1, unofficial] is from the project "Voxel-Terrain-Generation" by cadenmbond. The Binary Hidden-Face Culling variant is described in a devlog by Yunasawa for the project "YNL - Vozel" [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
