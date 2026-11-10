# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It sets itself apart from its siblings by operating on a per-voxel neighbor check basis during mesh creation, rather than using bitwise bulk operations [S2, unofficial], coverage masks [Sibling: Face Coverage Mask Culling], or post-generation visibility culling [Sibling: Block Visibility Culling].

## How it works
**Inputs and Outputs:**
*   **Input:** A data structure representing occupied voxel positions (e.g., a hash set of 3D integer coordinates or a bitmask array) [S1, unofficial][S2, unofficial].
*   **Output:** A list of visible faces (or a mesh buffer) containing only the exposed surfaces of the occupied voxels [S1, unofficial].

**Data Structures:**
The method relies on checking neighbor positions. A common implementation uses a hash set of 3D integer coordinates to represent occupied voxels [S1, unofficial].
*   `occupiedPositions`: A hash set containing the grid coordinates of all solid voxels.
*   `Cube` struct (optional, for per-voxel processing):
    *   `cubeCenter`: The grid position of the voxel.
    *   `faces`: A vector of size 6, one for each direction.
*   `Face` struct (optional):
    *   `vertices`: Vertex data.
    *   `normal`: Direction of the face.
    *   `isSideFace`: Boolean.
    *   `isBottomFace`: Boolean.
    *   `renderFlag`: Boolean (determines if the face is added to the mesh).

**Numbered Steps:**
1.  Iterate over every occupied voxel $P$ in the world grid.
2.  For each of the six immediate spatial neighbors $N$ (Up, Down, Left, Right, Front, Back):
    a. Calculate the neighbor position: $P_{neighbor} = P + N$.
    b. Check if a voxel exists at $P_{neighbor}$ in the occupancy data structure.
    c. Apply culling logic:
        *   If the neighbor exists, the face is hidden; do not add it.
        *   If the neighbor is empty/transparent, the face is exposed; add it.
        *   If the face is a bottom face, it is often culled entirely regardless of the neighbor below [S1, unofficial].

**Formulas:**
*   **Neighbor Position**:
    $$ P_{neighbor} = P + N $$
    Where $P$ is the current voxel's grid coordinate and $N$ is the direction vector (e.g., $(0, 1, 0)$ for Up).

*   **Basic Culling Condition (Opaque)**:
    $$ \text{Visible} = \neg \text{Occupied}(P_{neighbor}) $$
    Where $\text{Occupied}(P_{neighbor})$ is true if a voxel exists at $P_{neighbor}$.

*   **Bottom Face Culling**:
    $$ \text{BottomFace} \implies \text{Visible} = \text{false} $$
    Bottom faces are always culled to reduce geometry [S1, unofficial].

*   **Side Face Culling (Specific Implementation)**:
    In some implementations, side faces are also culled if a neighbor exists directly above the neighbor position, effectively hiding faces that are "shadowed" from above [S1, unofficial].
    $$ \text{Visible}_{side} = \neg \text{Occupied}(P_{neighbor}) \land \neg \text{Occupied}(P_{neighbor} + (0, 1, 0)) $$
    Note: This specific logic is derived from [S1, unofficial] and may vary by implementation. The standard approach is simply checking the immediate neighbor.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(occupied_positions):
    """
    Generates a mesh by culling hidden faces.
    
    Args:
        occupied_positions: A set of (x, y, z) tuples representing solid voxels.
        
    Returns:
        A list of visible faces, where each face is a tuple of vertices.
    """
    visible_faces = []
    directions = [
        (1, 0, 0),   # Right
        (-1, 0, 0),  # Left
        (0, 1, 0),   # Up
        (0, -1, 0),  # Down
        (0, 0, 1),   # Front
        (0, 0, -1)   # Back
    ]
    
    # Define face normals for reference (optional, for debugging or specific logic)
    face_normals = {
        (1, 0, 0): (1, 0, 0),
        (-1, 0, 0): (-1, 0, 0),
        (0, 1, 0): (0, 1, 0),
        (0, -1, 0): (0, -1, 0),
        (0, 0, 1): (0, 0, 1),
        (0, 0, -1): (0, 0, -1)
    }

    for voxel in occupied_positions:
        x, y, z = voxel
        
        # Check each of the 6 faces
        for dx, dy, dz in directions:
            neighbor = (x + dx, y + dy, z + dz)
            
            # Determine if this is a bottom face
            is_bottom = (dy == -1)
            
            # Bottom face culling: always culled
            if is_bottom:
                continue
            
            # Check if neighbor exists
            neighbor_exists = neighbor in occupied_positions
            
            # Basic culling: if neighbor exists, face is hidden
            if neighbor_exists:
                continue
                
            # Generate face geometry for the exposed side
            # Implementation detail: generate 2 triangles (quad) for this face
            face_vertices = create_face_geometry(x, y, z, dx, dy, dz)
            visible_faces.append(face_vertices)
            
    return visible_faces
```

**Edge Cases and Pitfalls:**
*   **Bottom Faces**: Bottom faces are always culled (`renderFlag = false`) regardless of neighbors below, as indicated by the logic `else { face.renderFlag = false; }` for `isBottomFace` [S1, unofficial].
*   **Side Face Logic**: Side faces (non-bottom) are culled if a neighbor exists directly adjacent OR if a neighbor exists directly above the neighbor position (checking `neighborPos + glm::ivec3(0, 1, 0)`) [S1, unofficial]. This is a specific optimization/pitfall noted in [S1, unofficial].
*   **Performance**: The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial]. Using a hash set or contiguous array improves lookup speed.

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk. It sets itself apart from standard Face Culling by avoiding per-voxel neighbor lookups in favor of bulk bitwise shifts and masks, offering better performance for dense grids. Choose this when working with contiguous 1D arrays or bitmasks and seeking maximum throughput for simple opaque voxel culling. Link: [Binary Hidden-Face Culling](2176-voxel-block-shape-culling-configuration.md) (Note: No dedicated article exists, but it is a variant of the family).

## Operating characteristics
*   **Time Complexity:** $O(V \cdot F)$, where $V$ is the number of occupied voxels and $F$ is the number of faces per voxel (typically 6). Each voxel requires 6 neighbor checks.
*   **Memory Complexity:** $O(V)$ to store the occupied positions and the resulting mesh.
*   **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
*   **Strengths:** Simple to implement; reduces geometry significantly for sparse or structured scenes; no framerate loss during rendering as culling runs once during mesh creation [S1, unofficial].
*   **Limitations:** Naive implementation with random memory access (e.g., hash sets) can be slow for large worlds [S2, unofficial]. Bottom face culling is hardcoded in some implementations, which may not be desired for all use cases (e.g., floating islands).
*   **When to use:** When generating static voxel meshes and reducing draw calls is a priority.
*   **When not to use:** When dynamic voxel changes require frequent mesh regeneration and performance is critical (consider Binary Hidden-Face Culling or chunk-based approaches).

## Background
Face Culling is a fundamental technique in voxel terrain generation, originating from the need to optimize the rendering of block-based worlds like Minecraft. It was popularized by early voxel engines and remains a standard step in mesh generation pipelines. The specific implementation details, such as bottom-face culling and side-face shadowing, are documented in various community devlogs and open-source projects [S1, unofficial][S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
