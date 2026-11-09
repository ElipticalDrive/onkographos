# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices and triangles sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, which results in excessive draw calls and vertex processing overhead. It sets itself apart from siblings like [Block Visibility Culling](5334-block-visibility-configuration.md) (which operates on larger chunks) and [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md) (which considers material transparency) by focusing strictly on geometric adjacency and immediate neighbor occupancy to eliminate internal faces during the static mesh construction phase [S1, unofficial].

## How it works
The method iterates through voxel positions and determines face visibility based on the occupancy state of immediate neighbors. A common implementation uses a hash set or grid lookup for $O(1)$ neighbor checks [8735-face-culling.md].

### Inputs and Outputs
*   **Input:** A collection of voxel positions (typically stored in a hash set or grid) representing solid voxels.
*   **Output:** A vertex buffer containing only the exposed faces of the voxel mesh.

### Data Structures
The implementation relies on specific data structures to track voxel state and face properties [S1, unofficial]:

1.  **Occupied Positions:** A hash set (`std::unordered_set<glm::ivec3>`) or a dense grid lookup table storing the integer coordinates of all solid voxels. This allows for constant-time existence checks.
2.  **Cube Struct:** Represents a single voxel unit.
    *   `std::vector<Face> faces`: An array of size 6, one for each potential face (±X, ±Y, ±Z).
    *   `glm::ivec3 cubeCenter`: The integer grid coordinates $(x, y, z)$ of the voxel.
    *   `glm::ivec3 gridPos`: Redundant or auxiliary position data.
    *   `int neighbors`: A counter for adjacent solid voxels (used for auxiliary calculations like ambient occlusion or structural integrity, though not strictly required for basic culling).
3.  **Face Struct:** Represents a single quad face of the voxel.
    *   `std::vector<float> vertices`: The 8 vertices (or 6 for a triangle strip) defining the quad geometry.
    *   `glm::ivec3 normal`: The outward-facing normal vector of the face (e.g., $(1, 0, 0)$ for the +X face).
    *   `bool isSideFace`: Boolean flag indicating if the face is a vertical side face (normal has $Y=0$).
    *   `bool isBottomFace`: Boolean flag indicating if the face is the bottom face (normal is $(0, -1, 0)$).
    *   `bool renderFlag`: Boolean flag determining if the face should be included in the final mesh.

### Algorithm Steps
The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

1.  **Initialize:** For each voxel at position $P$, initialize all 6 faces with `renderFlag = true`.
2.  **Evaluate Bottom Faces:** For the face with normal $(0, -1, 0)$, set `renderFlag = false`. Bottom faces are always culled to hide the underside of terrain, regardless of whether there is a voxel below [8735-face-culling.md].
3.  **Evaluate Side and Top Faces:** For each remaining face with normal $N_{face}$:
    *   Calculate the neighbor position: $P_{neighbor} = P + N_{face}$.
    *   Check if a solid voxel exists at $P_{neighbor}$.
    *   If a neighbor exists at $P_{neighbor}$, the face is internal and `renderFlag` is set to `false`.
    *   If no neighbor exists at $P_{neighbor}$, the face is potentially exposed. However, for side faces, an additional check is performed to handle overhangs:
        *   Calculate the position directly above the neighbor: $P_{above\_neighbor} = P_{neighbor} + (0, 1, 0)$.
        *   If a solid voxel exists at $P_{above\_neighbor}$, the face is occluded by the overhang, and `renderFlag` is set to `false`.
        *   Otherwise, `renderFlag` remains `true`.
4.  **Generate Mesh:** Iterate through all voxels and their faces. If `renderFlag` is `true`, add the face's vertices to the vertex buffer.

### Pseudocode
The following pseudocode implements the logic described in [S1, unofficial]. It assumes a `Cube` class and a global `occupiedPositions` set.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cull_faces(cube: Cube, occupied_positions: set):
    """
    Determines which faces of a voxel should be rendered based on neighbor occupancy.
    
    Args:
        cube: The voxel object containing its position and face definitions.
        occupied_positions: A set of (x, y, z) tuples representing solid voxels.
    """
    for face in cube.faces:
        # Calculate the position of the immediate neighbor in the direction of the face normal
        neighbor_pos = cube.cubeCenter + face.normal
        
        # Check if a solid voxel exists at the immediate neighbor position
        neighbor_exists = neighbor_pos in occupied_positions
        
        if face.isBottomFace:
            # Bottom faces are always culled to hide the underside of terrain
            face.renderFlag = False
        else:
            # For side and top faces, check for direct adjacency
            if neighbor_exists:
                # Face is hidden by an adjacent voxel
                face.renderFlag = False
            else:
                # Check for overhangs: if a voxel exists directly above the neighbor position,
                # the current face is occluded by the overhang
                above_neighbor_pos = neighbor_pos + (0, 1, 0)
                if above_neighbor_pos in occupied_positions:
                    face.renderFlag = False
                else:
                    # Face is exposed
                    face.renderFlag = True

        # Optional: Count neighbors for auxiliary data (e.g., ambient occlusion)
        # This loop checks the 6 horizontal neighbors of the voxel above the current one
        # to count how many solid voxels are adjacent to the top layer.
        if not face.isBottomFace:
            above_offsets = [
                (1, 1, 0), (1, 1, 1), (1, 1, -1),
                (-1, 1, 0), (-1, 1, -1), (-1, 1, 1)
            ]
            for offset in above_offsets:
                if (cube.cubeCenter + offset) in occupied_positions:
                    cube.neighbors += 1
```

### Parameters and Edge Cases
*   **Bottom Face Culling:** The algorithm assumes that the bottom face of any voxel is never visible. This is a common optimization in terrain generation to reduce vertex count by 1/6th. If underground visibility is required, this logic must be modified to check for neighbors below [S1, unofficial].
*   **Overhang Handling:** The check for `neighbor_pos + (0, 1, 0)` ensures that side faces are culled even if the immediate neighbor is missing, provided there is a voxel directly above that neighbor. This correctly handles overhangs where a voxel is supported by a voxel above it rather than below it [8735-face-culling.md].
*   **Precision:** The algorithm relies on integer grid coordinates. Floating-point inaccuracies are not a factor if the voxel grid is aligned to integer boundaries.
*   **Degenerate Input:** If the `occupied_positions` set is empty, no faces are rendered. If it contains a single voxel, only the top and 4 side faces are rendered (bottom is culled).

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart from the neighbor-check method by operating on linear arrays of bits rather than 3D spatial lookups, allowing for vectorized or bulk processing. It is chosen when the voxel data can be flattened into a 1D array (e.g., along a single axis for side faces) and performance is critical. The method represents each voxel as a bit in a bitmask (1 = solid, 0 = transparent/air). To determine visible left faces, the bitmask is shifted right by 1 position (representing the neighbor to the left), inverted to find air positions, and bitwise ANDed with the original mask. Visible right faces are determined by shifting left. This avoids random memory access and looping through neighbors individually, which leads to poor performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V)$ where $V$ is the number of voxels, assuming $O(1)$ hash set lookups. Each voxel requires 6 lookups.
*   **Memory Complexity:** $O(V)$ to store the occupied positions and the resulting mesh.
*   **Measured Performance:** In a specific implementation, 4096 cubes were rendered at ~60 FPS on the author's system [S1, unofficial].
*   **Strengths:** Eliminates internal faces entirely, significantly reducing vertex count and draw calls. The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].
*   **Limitations:** Requires a full mesh rebuild if the voxel data changes. The neighbor-check method may suffer from cache misses if the `occupied_positions` set is sparse and uses a hash table. The binary method requires flattening data and may be less intuitive for 3D spatial reasoning.
*   **When to use:** When generating static or semi-static voxel terrain where the mesh does not change frequently.
*   **When not to use:** When the voxel data changes every frame (dynamic voxel destruction/placement) and the overhead of rebuilding the mesh outweighs the rendering benefits.

## Background
Face Culling is a fundamental optimization in voxel-based game development and 3D modeling. It originated from the need to render large voxel worlds efficiently, as naively rendering all faces of every voxel results in an unmanageable number of triangles. The specific implementation details described here, including the handling of overhangs and bottom-face culling, are documented in community devlogs and open-source projects [S1, S2, unofficial]. The binary variant was popularized by developers seeking to optimize mesh generation performance by leveraging bitwise operations [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
