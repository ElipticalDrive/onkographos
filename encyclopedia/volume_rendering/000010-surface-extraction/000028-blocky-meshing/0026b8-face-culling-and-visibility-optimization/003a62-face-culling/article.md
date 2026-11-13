# Face Culling

Also called: Exposure Culling, Face Occlusion Culling, Neighbor Check Culling, Only render block faces that aren't adjacent to another face, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that prevents rendering block faces occluded by adjacent solid voxels, significantly reducing triangle count in homogeneous voxel terrain [S1, unofficial][S2, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, by exposing only faces adjacent to empty space or out-of-bounds [S2, unofficial]. Unlike [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) which may handle placement logic, or [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) which uses coverage masks, this method relies on direct neighbor occupancy checks during mesh construction [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of solid voxel positions, typically stored in a hash set or grid structure.
*   **Output:** A vertex buffer containing only the exposed faces of the voxel structure.

**Data Structures:**
The implementation may use a `Cube` struct to manage face state [S2, unofficial]:
*   `std::vector<Face> faces`: Array of size 6, one for each face direction.
*   `glm::ivec3 cubeCenter`: The 3D integer coordinates of the voxel.
*   `glm::ivec3 gridPos`: Grid position (often same as cubeCenter).
*   `int neighbors`: Count of adjacent solid voxels.

Each `Face` struct contains [S2, unofficial]:
*   `std::vector<float> vertices`: Vertex data for the face.
*   `glm::ivec3 normal`: The normal vector of the face.
*   `bool isSideFace`: Boolean indicating if the face is a side (non-bottom) face.
*   `bool isBottomFace`: Boolean indicating if the face is the bottom face.
*   `bool renderFlag`: Boolean determining if the face is added to the buffer.

**Algorithm Steps:**
1.  Iterate through each solid voxel in the world.
2.  For each of the 6 faces of the voxel, determine the position of the immediate neighbor in the direction of the face normal: $P_{neighbor} = P_{voxel} + N$, where $N$ is the face normal vector.
3.  Check if a solid voxel exists at $P_{neighbor}$ [S1, unofficial][S2, unofficial].
4.  If a neighbor exists, the face is internal and is culled (not added to the vertex buffer) [S1, unofficial][S2, unofficial].
5.  If no neighbor exists (the neighbor is empty/air or out of bounds), the face is exposed and is added to the vertex buffer [S2, unofficial].
6.  This check runs once during mesh creation, resulting in no framerate loss during rendering [S2, unofficial].

**Pseudocode:**
The following pseudocode implements the neighbor check logic described in [S2, unofficial], including specific handling for bottom and side faces as noted in the source.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(occupied_positions):
    """
    Generates a mesh by culling faces adjacent to solid neighbors.
    occupied_positions: A set of glm::ivec3 coordinates representing solid voxels.
    Returns: A list of vertices for exposed faces.
    """
    mesh_vertices = []
    
    # Define face normals for the 6 directions
    # Order: +X, -X, +Y, -Y, +Z, -Z
    # Note: Source [S2, unofficial] distinguishes between 'bottom' and 'side' faces.
    # We assume +Y is Up, so -Y is Bottom.
    face_normals = [
        (1, 0, 0),   # Right
        (-1, 0, 0),  # Left
        (0, 1, 0),   # Top
        (0, -1, 0),  # Bottom
        (0, 0, 1),   # Front
        (0, 0, -1)   # Back
    ]
    
    for voxel_pos in occupied_positions:
        for i, normal in enumerate(face_normals):
            # Determine neighbor position
            neighbor_pos = voxel_pos + normal
            
            # Determine if this is a bottom face
            is_bottom_face = (normal == (0, -1, 0))
            
            # Determine if this is a side face (non-bottom)
            is_side_face = not is_bottom_face
            
            render_flag = False
            
            if is_bottom_face:
                # Source [S2, unofficial] pitfall: Bottom faces are always culled
                # regardless of neighbors below.
                render_flag = False
            else:
                # Check for direct neighbor
                neighbor_exists = neighbor_pos in occupied_positions
                
                if is_side_face:
                    # Source [S2, unofficial] pitfall: Side faces are culled if:
                    # 1. A neighbor exists directly adjacent, OR
                    # 2. A neighbor exists directly above the neighbor position.
                    # This implies checking for overhangs or specific geometry rules.
                    if neighbor_exists:
                        render_flag = False
                    else:
                        # Check position above the neighbor
                        above_neighbor_pos = neighbor_pos + (0, 1, 0)
                        if above_neighbor_pos in occupied_positions:
                            render_flag = False
                        else:
                            render_flag = True
                else:
                    # Fallback for non-bottom, non-side (e.g. Top face)
                    # Standard logic: render if no neighbor
                    render_flag = not neighbor_exists
            
            if render_flag:
                # Add face vertices to mesh_vertices
                # Implementation detail: [G] Add the 4 vertices and 2 indices 
                # for the quad defined by voxel_pos and normal.
                add_face_to_mesh(mesh_vertices, voxel_pos, normal)
                
    return mesh_vertices
```

**Parameters and Edge Cases:**
*   **Chunk Boundaries:** Adjacent cubes across chunk boundaries (e.g., x=0 and x=15 in a 16-wide chunk) must be checked. If ignored, the outline of each chunk is drawn, killing performance [S1, unofficial].
*   **Bottom Face Culling:** In the specific implementation from [S2, unofficial], bottom faces are always culled (`renderFlag = false`) regardless of neighbors below. This is a specific design choice, not a universal rule for all face culling methods.
*   **Side Face Logic:** The source [S2, unofficial] specifies that side faces are also culled if a neighbor exists *above* the neighbor position. This suggests a specific geometric constraint or optimization for the referenced project, differing from standard "exposed to air" logic. Implementers should verify if this overhang check is required for their specific voxel rules.

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S3, unofficial]. It represents each voxel as a bit in a bitmask (1 = solid, 0 = transparent/air) [S3, unofficial]. To determine visible left faces, it shifts the bitmask right by 1 position, inverts the result, and performs a bitwise AND with the original mask. Right faces use a left shift [S3, unofficial]. This approach avoids the poor performance of looping through neighbors and performing random memory access [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V)$, where $V$ is the number of solid voxels. Each voxel requires 6 neighbor lookups.
*   **Memory Complexity:** $O(F)$, where $F$ is the number of exposed faces. Significantly less than $6V$.
*   **Performance:** With this optimization, performance allows running more than 14,440 chunks on a fairly old PC (Intel Core i7 2.5 GHz, AMD Radeon HD 6770M 1024 MB) while performing GPU-heavy algorithms like cascaded shadow mapping and water shading [S1, unofficial]. Without it, the author could only draw ~20 chunks [S1, unofficial]. One implementation reported 4096 cubes rendered at ~60 FPS [S2, unofficial].
*   **Strengths:** Dramatically reduces triangle count (drawing only ~1/1000 of triangles previously drawn in homogeneous terrain) [S1, unofficial]. No runtime overhead as culling happens during mesh generation [S2, unofficial].
*   **Limitations:** Requires efficient neighbor lookup structures (e.g., hash sets) to avoid $O(V^2)$ complexity. Chunk boundary checks are critical to avoid visual artifacts and performance hits [S1, unofficial].

## Background
Face Culling is a standard optimization in voxel rendering, widely discussed in game development communities. It is cited as the most important optimization for voxel performance by developers who achieved massive improvements by ensuring it works on all cases, including chunk limits [S1, unofficial]. It is used in voxel game implementations alongside VBO per chunk, frustum culling, and backface culling [S1, unofficial].

## Sources
[S1] Optimising Voxel World Rendering, https://gamedev.stackexchange.com/questions/117398/optimising-voxel-world-rendering, fetched 2026-10-27 (unofficial)
[S2] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S3] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
