# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices and indices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It sets itself apart from its siblings by performing this check primarily through direct neighbor lookup during mesh creation, rather than using coverage masks, backface culling, or opacity-based thresholds.

## How it works
The method determines face visibility by evaluating voxel adjacency. For each voxel, the algorithm checks the occupancy of its six immediate neighbors. A face is generated if and only if the neighbor in that direction is empty (air) or transparent. If the neighbor is solid/opaque, the face is internal and culled.

### Inputs and Outputs
- **Input:** A set of occupied voxel positions, typically represented as a hash set of integer coordinates or a dense grid.
- **Output:** A list of visible faces (vertices and indices) to be added to the mesh buffer.

### Data Structures
The implementation often uses specific structures to manage voxel state and face properties [S1, unofficial]:

1.  **`Cube` struct:**
    -   `faces`: A vector of 6 `Face` objects, one for each direction ($\pm X, \pm Y, \pm Z$).
    -   `cubeCenter`: The 3D integer coordinates of the voxel center (`glm::ivec3`).
    -   `gridPos`: The grid position of the voxel (`glm::ivec3`).
    -   `neighbors`: An integer counter for adjacent solid voxels.

2.  **`Face` struct:**
    -   `vertices`: A vector of floats representing the geometric vertices of the face.
    -   `normal`: The 3D integer normal vector of the face (`glm::ivec3`).
    -   `isSideFace`: Boolean flag indicating if the face is a side face (not top/bottom).
    -   `isBottomFace`: Boolean flag indicating if the face is the bottom face.
    -   `renderFlag`: Boolean flag determining if the face should be rendered.

### Algorithm Steps
The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

1.  Iterate through each voxel in the world grid.
2.  For each of the 6 faces of the voxel:
    a.  Calculate the position of the neighbor in the direction of the face's normal: $P_{neighbor} = P_{center} + \vec{n}$.
    b.  Check if a voxel exists at $P_{neighbor}$ in the occupied positions set.
    c.  Apply visibility logic based on face type:
        -   **Bottom Faces:** Always culled (`renderFlag = false`), regardless of neighbors below [S1, unofficial].
        -   **Side Faces:**
            -   If a neighbor exists directly adjacent ($P_{neighbor}$ is occupied), culled.
            -   If a neighbor exists directly above the neighbor position ($P_{neighbor} + (0,1,0)$ is occupied), culled. This handles cases where a higher voxel overhangs the side face.
            -   Otherwise, visible (`renderFlag = true`).
3.  Collect all faces with `renderFlag = true` into the mesh buffer.

### Pseudocode
The following pseudocode implements the neighbor check logic described in [S1, unofficial]. It assumes a `Cube` object with a list of `Face` objects, each having a `normal` vector.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cull_faces(cube: Cube, occupied_positions: set):
    """
    Determines which faces of a cube should be rendered based on neighbor occupancy.
    
    Args:
        cube: The voxel cube object containing faces and center position.
        occupied_positions: A set of (x, y, z) tuples representing solid voxels.
    """
    center = cube.cubeCenter
    
    for face in cube.faces:
        # Calculate neighbor position in the direction of the face normal
        neighbor_pos = center + face.normal
        
        # Check if neighbor exists
        neighbor_exists = neighbor_pos in occupied_positions
        
        # Determine render flag based on face type
        if face.isBottomFace:
            # Bottom faces are always culled
            face.renderFlag = False
        else:
            # For side faces, check direct neighbor and the voxel above it
            # Check direct neighbor
            if neighbor_exists:
                face.renderFlag = False
            else:
                # Check if there is a voxel directly above the neighbor position
                # This handles overhangs where a higher voxel blocks the side face
                above_neighbor_pos = neighbor_pos + (0, 1, 0)
                if above_neighbor_pos in occupied_positions:
                    face.renderFlag = False
                else:
                    face.renderFlag = True

    # Optional: Count neighbors for other purposes (e.g., AO)
    # Note: The source code counts neighbors in a specific way involving offsets
    # relative to the cube center, but this is not strictly part of the culling
    # visibility logic itself.
```

### Parameters and Edge Cases
-   **Bottom Face Culling:** The method explicitly culls all bottom faces (`isBottomFace == true`) regardless of the occupancy below. This is a design choice, possibly to support floating terrain or specific rendering optimizations [S1, unofficial].
-   **Overhang Handling:** Side faces are culled not only if a neighbor exists at the same level but also if a neighbor exists directly above the neighbor's position. This prevents rendering side faces that are occluded by overhanging voxels [S1, unofficial].
-   **Performance:** The check is performed once during mesh generation. The source reports rendering 4096 cubes at ~60 FPS on the author's system [S1, unofficial].

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart by processing entire rows or columns of voxels simultaneously using bitwise shifts and masks, avoiding per-voxel loops and random memory access [S2, unofficial]. To determine visible left faces, shift the bitmask right by 1 position (representing the neighbor to the left), invert the result to find air positions, and perform a bitwise AND with the original mask: $M_{left} = M \ \& \ \sim(M \gg 1)$. Similarly, for right faces, shift left: $M_{right} = M \ \& \ \sim(M \ll 1)$. Pitfall: The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel's 6 faces are checked once.
-   **Memory Complexity:** $O(N)$ to store the voxel grid and the resulting mesh.
-   **Performance:** The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial]. Measured performance: 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
-   **Strengths:** Simple to implement; effectively reduces vertex count by removing internal faces; no runtime overhead.
-   **Limitations:** Does not handle overhangs or complex occlusions beyond immediate neighbors unless additional checks (like the "above neighbor" check) are added; bottom faces are always culled which may not be desired for all scenes.
-   **When to use:** When generating static or semi-static voxel meshes where internal faces are the primary source of excess geometry.
-   **When not to use:** When dynamic occlusion from distant objects is required, or when bottom faces must be visible.

## Background
Face Culling is a standard optimization in voxel terrain generation, often cited in early voxel engine tutorials and implementations. It is a fundamental step in converting voxel data into a renderable mesh, preventing the GPU from processing faces that are never visible. The specific implementation details, such as the `Cube` and `Face` structs and the overhang handling logic, are documented in community-driven projects like the Voxel-Terrain-Generation repository [S1, unofficial]. The binary variant using bitmasks was popularized in devlogs for voxel games like YNL - Vozel to address performance issues with naive neighbor checking [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
