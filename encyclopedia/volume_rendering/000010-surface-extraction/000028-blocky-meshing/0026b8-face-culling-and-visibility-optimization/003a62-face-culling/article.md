# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors [S1, unofficial]. It sets itself apart from its siblings by performing visibility checks during the static mesh creation phase rather than at runtime or via complex projection, relying on immediate neighbor occupancy states to determine face visibility [Parent].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of occupied voxel positions in a 3D grid, typically represented as a hash set or bitmask.
*   **Output:** A vertex buffer containing only the exposed faces of the voxels.

**Data Structures:**
The implementation typically reconstructs cubes from triangles rather than instantiating pre-made models [S1, unofficial]. To optimize performance, the meshes of all individual cubes are often combined into a single big mesh called a 'chunk' [S1, unofficial].

A common data layout for a voxel object includes:
*   `Cube` struct:
    *   `faces`: A list of 6 `Face` objects, one for each direction.
    *   `cubeCenter`: The 3D integer coordinates of the voxel center.
    *   `gridPos`: The grid position of the voxel.
    *   `neighbors`: An integer counter for adjacent occupied voxels.
*   `Face` struct:
    *   `vertices`: A list of float values representing the geometric vertices of the face.
    *   `normal`: The 3D integer vector indicating the face's outward direction.
    *   `isSideFace`: A boolean indicating if the face is a vertical side.
    *   `isBottomFace`: A boolean indicating if the face is the bottom.
    *   `renderFlag`: A boolean determining if the face should be added to the mesh.

**Algorithm Steps:**
The culling check is performed once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

1.  Iterate through each occupied voxel in the world grid.
2.  For each voxel, examine its six adjacent positions in the world grid.
3.  For each of the six faces of the current voxel:
    a.  Determine the position of the neighbor in the direction of the face's normal:
        $$ P_{neighbor} = P_{current} + N_{face} $$
        where $P_{current}$ is the current voxel's position and $N_{face}$ is the face's normal vector.
    b.  Check if a voxel exists at $P_{neighbor}$ in the occupied positions set.
    c.  Apply visibility logic based on face type and neighbor presence:
        *   **Bottom Faces:** Always culled. Set `renderFlag` to `false` regardless of neighbors below [S1, unofficial].
        *   **Side Faces:** A face is visible only if it is exposed. A side face is culled if:
            1.  A neighbor exists directly adjacent in the face's direction ($P_{neighbor}$ is occupied).
            2.  A neighbor exists directly above the neighbor position ($P_{neighbor} + (0, 1, 0)$ is occupied). This specific logic prevents rendering side faces that are occluded by overhangs or adjacent blocks above the neighbor [S1, unofficial].
            If neither condition is met, the face is exposed and `renderFlag` is set to `true`.
4.  Collect all faces with `renderFlag` set to `true` into the vertex buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mesh(occupied_positions: set[tuple[int, int, int]]) -> list[Vertex]:
    """
    Generates a mesh by culling internal faces of voxels.
    occupied_positions: A set of (x, y, z) tuples representing solid voxels.
    Returns: A list of vertices for the visible faces.
    """
    vertices = []
    
    # Define the 6 directions and their corresponding face normals
    # Order: +X, -X, +Y, -Y, +Z, -Z
    directions = [
        (1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)
    ]
    
    # Helper to generate vertices for a unit cube face
    # This is a placeholder for the specific geometry generation logic
    def get_face_vertices(center: tuple[int, int, int], normal: tuple[int, int, int]) -> list[float]:
        # [G] Standard unit cube face generation based on normal.
        # Returns 4 vertices (or 6 for triangles) relative to center.
        pass 

    for pos in occupied_positions:
        x, y, z = pos
        
        for i, normal in enumerate(directions):
            # Calculate neighbor position
            nx, ny, nz = x + normal[0], y + normal[1], z + normal[2]
            neighbor_exists = (nx, ny, nz) in occupied_positions
            
            # Determine if this face is visible
            visible = False
            
            # Check for bottom face (assuming Y is up, -Y is bottom)
            # In the source, isBottomFace logic always culls.
            # Assuming index 3 corresponds to -Y (0, -1, 0) based on typical ordering,
            # but the source code checks `isBottomFace` property.
            # We will implement the logic described: Bottom faces are always culled.
            if normal == (0, -1, 0):
                visible = False
            else:
                # Side faces (and potentially top, though source focuses on side/bottom)
                # Logic: Cull if neighbor exists directly OR if neighbor exists above the neighbor
                
                # Check direct neighbor
                direct_occlusion = neighbor_exists
                
                # Check above-neighbor occlusion (only for side faces in source logic)
                # Source checks: occupiedPositions.find(neighborPos + glm::ivec3(0, 1, 0))
                above_neighbor_pos = (nx, ny + 1, nz)
                above_occlusion = above_neighbor_pos in occupied_positions
                
                if direct_occlusion or above_occlusion:
                    visible = False
                else:
                    visible = True
            
            if visible:
                # Add vertices for this face
                face_verts = get_face_vertices(pos, normal)
                vertices.extend(face_verts)
                
    return vertices
```

**Parameters and Edge Cases:**
*   **Bottom Face Culling:** The source implementation explicitly culls all bottom faces (`renderFlag = false`) regardless of whether a voxel exists below [S1, unofficial]. This is a specific design choice, likely for terrain rendering where the ground is not visible from below.
*   **Above-Neighbor Check:** The source includes a check for neighbors directly above the adjacent voxel (`neighborPos + (0, 1, 0)`) for side faces [S1, unofficial]. This handles cases where a side face might be visually occluded by a block sitting on top of the adjacent block.
*   **Precision:** The algorithm relies on integer grid coordinates. Floating-point precision issues are not a factor in the culling logic itself, only in the subsequent rendering pipeline.

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart from standard neighbor checks by processing entire rows or columns of voxels simultaneously using bitwise shifts and masks, avoiding per-voxel loops and random memory access [S2, unofficial]. Choose this method when performance is critical and the voxel data can be efficiently mapped to linear bitmasks (e.g., single-axis slices).
*   **Definition:** Uses bitwise operations on a bitmask where 1 indicates a solid voxel and 0 indicates transparent/air [S2, unofficial].
*   **Steps:**
    1.  Represent the voxel layer as a bitmask $M$.
    2.  To find visible left faces: Shift $M$ right by 1 ($M >> 1$) to represent neighbors to the left. Invert the result ($\sim(M >> 1)$) to find air positions. Perform bitwise AND with the original mask: $M \ \& \ \sim(M >> 1)$.
    3.  To find visible right faces: Shift $M$ left by 1 ($M << 1$). Invert the result ($\sim(M << 1)$). Perform bitwise AND: $M \ \& \ \sim(M << 1)$.
    4.  Use the resulting masks to determine which faces to draw [S2, unofficial].
*   **Pitfall:** The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V)$, where $V$ is the number of occupied voxels. Each voxel is processed once, and neighbor lookups are $O(1)$ on average for hash sets.
*   **Memory Complexity:** $O(F)$, where $F$ is the number of visible faces. The output mesh size is proportional to the surface area of the voxel volume.
*   **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
*   **Strengths:** No framerate loss during rendering because culling is done once during mesh creation [S1, unofficial]. Simple to implement.
*   **Limitations:** Bottom faces are always culled in the reference implementation, which may not be suitable for all use cases (e.g., underground exploration) [S1, unofficial]. The "above-neighbor" check for side faces may be overly conservative or specific to certain terrain types [S1, unofficial].
*   **When to use:** Static voxel terrain where the mesh is generated infrequently.
*   **When not to use:** Dynamic voxel worlds where voxels change frequently, as the mesh would need to be regenerated.

## Background
Face Culling in voxel rendering is a standard optimization technique in procedural terrain generation. The specific implementation details described here originate from a GitHub repository for voxel terrain generation [S1, unofficial]. The binary variant was documented in a devlog for a Unity-based voxel project [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
