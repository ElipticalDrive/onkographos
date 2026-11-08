# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization for blocky voxel meshes that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors [S1, unofficial]. It differs from its siblings by performing a direct per-voxel neighbor check during mesh assembly to determine face visibility, rather than using coverage masks, backface culling, or opacity-based thresholds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of occupied voxel positions in a 3D grid, typically represented as a hash set or grid lookup structure.
*   **Output:** A vertex buffer containing only the exposed faces of the voxels.

**Data Structures:**
The method relies on specific data structures to track voxel state and face properties [S1, unofficial]:
*   `Cube` struct:
    *   `std::vector<Face> faces`: A vector of size 6, one for each face of the cube.
    *   `glm::ivec3 cubeCenter`: The 3D integer coordinates of the voxel's center.
    *   `glm::ivec3 gridPos`: The grid position of the voxel.
    *   `int neighbors`: A counter for adjacent neighbors (used for specific culling logic).
*   `Face` struct:
    *   `std::vector<float> vertices`: The vertex data for the face.
    *   `glm::ivec3 normal`: The normal vector of the face.
    *   `bool isSideFace`: Boolean flag indicating if the face is a side face.
    *   `bool isBottomFace`: Boolean flag indicating if the face is a bottom face.
    *   `bool renderFlag`: Boolean flag determining if the face should be rendered.

**Algorithm Steps:**
1.  **Neighbor Check:** For each voxel at position $P_{current}$, iterate through its six adjacent positions in the world grid [S1, unofficial].
2.  **Visibility Determination:**
    *   Calculate the neighbor position $P_{neighbor} = P_{current} + \mathbf{n}$, where $\mathbf{n}$ is the normal vector of the face being checked [S1, unofficial].
    *   Check if a solid voxel exists at $P_{neighbor}$ using the occupied positions set [S1, unofficial].
    *   **Bottom Faces:** If the face is a bottom face (`isBottomFace` is true), set `renderFlag` to `false` (culled) regardless of neighbors below [S1, unofficial].
    *   **Side Faces:** If the face is a side face (`isBottomFace` is false):
        *   Check if a neighbor exists directly adjacent at $P_{neighbor}$.
        *   Check if a neighbor exists directly above the neighbor position at $P_{neighbor} + (0, 1, 0)$ [S1, unofficial].
        *   If a neighbor exists at $P_{neighbor}$ OR at $P_{neighbor} + (0, 1, 0)$, set `renderFlag` to `false` [S1, unofficial].
        *   Otherwise, set `renderFlag` to `true` (exposed) [S1, unofficial].
3.  **Mesh Assembly:** Combine the meshes of all individual cubes into a single big mesh called a 'chunk' to optimize implementation [S1, unofficial]. Only faces with `renderFlag` set to `true` are added to the vertex buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mesh(occupied_positions: set[Vector3i]) -> list[Face]:
    """
    Generates a voxel mesh using face culling.
    
    Args:
        occupied_positions: A set of 3D integer coordinates representing solid voxels.
        
    Returns:
        A list of Face objects that are exposed and should be rendered.
    """
    mesh_faces = []
    
    # Define the 6 possible face normals
    normals = [
        Vector3i(1, 0, 0),  # Right
        Vector3i(-1, 0, 0), # Left
        Vector3i(0, 1, 0),  # Top
        Vector3i(0, -1, 0), # Bottom
        Vector3i(0, 0, 1),  # Front
        Vector3i(0, 0, -1)  # Back
    ]
    
    for cube_center in occupied_positions:
        for normal in normals:
            # Calculate neighbor position
            neighbor_pos = cube_center + normal
            
            # Determine if the face is a bottom face
            # Note: In the source, 'isBottomFace' is a property of the face.
            # Here we infer it from the normal direction for implementation clarity.
            is_bottom_face = (normal == Vector3i(0, -1, 0))
            
            render_flag = False
            
            if is_bottom_face:
                # Bottom faces are always culled regardless of neighbors below
                render_flag = False
            else:
                # Side faces (and potentially top, though logic focuses on side/bottom distinction)
                # Check for direct neighbor
                neighbor_exists = neighbor_pos in occupied_positions
                
                # Check for neighbor directly above the neighbor position
                # This specific logic is cited from the source code snippet
                above_neighbor_pos = neighbor_pos + Vector3i(0, 1, 0)
                above_neighbor_exists = above_neighbor_pos in occupied_positions
                
                if neighbor_exists or above_neighbor_exists:
                    render_flag = False
                else:
                    render_flag = True
            
            if render_flag:
                # Create face geometry
                face = create_face_geometry(cube_center, normal)
                mesh_faces.append(face)
                
    return mesh_faces

def create_face_geometry(center: Vector3i, normal: Vector3i) -> Face:
    """
    Helper function to generate vertex data for a face.
    Implementation details for vertex generation are not provided in the source.
    """
    # [G] Standard voxel face generation logic would go here.
    # This function is a placeholder for the actual vertex buffer population.
    return Face(vertices=[], normal=normal, is_bottom_face=(normal == Vector3i(0, -1, 0)))
```

**Parameters and Edge Cases:**
*   **Bottom Face Culling:** The method explicitly culls all bottom faces (`renderFlag = false`) regardless of whether there is a voxel below [S1, unofficial]. This is a specific design choice of this implementation.
*   **Side Face Logic:** Side faces are culled if there is a neighbor directly adjacent OR if there is a neighbor directly above the adjacent neighbor position [S1, unofficial]. This specific "above neighbor" check is a distinctive feature of this variant.
*   **Performance:** The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It differs from standard neighbor checks by operating on bulk bitmasks rather than iterating individual voxels and neighbors. To determine visible left faces, shift the bitmask right by 1 position (representing the neighbor to the left), invert the result to find air positions, and perform a bitwise AND with the original mask: $VisibleLeft = Mask \ \& \ \sim(Mask \gg 1)$ [S2, unofficial]. Similarly, for right faces, shift left: $VisibleRight = Mask \ \& \ \sim(Mask \ll 1)$ [S2, unofficial]. Use the resulting masks to choose which faces to draw [S2, unofficial]. Pitfall: The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The algorithm iterates through each occupied voxel and checks its six neighbors. The complexity is $O(V)$, where $V$ is the number of occupied voxels, assuming $O(1)$ lookup in the occupied positions set [S1, unofficial].
*   **Memory Complexity:** Requires storage for the occupied positions set and the resulting mesh vertices. The `Cube` struct stores neighbor counts and face flags [S1, unofficial].
*   **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
*   **Strengths:** Reduces the number of vertices sent to the GPU by skipping hidden faces [S1, unofficial]. The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].
*   **Limitations:** The specific logic for side faces (checking for neighbors above neighbors) may not be standard for all voxel engines and could lead to incorrect culling in some scenarios if not intended [S1, unofficial]. Bottom faces are always culled, which may not be desired for all applications [S1, unofficial].
*   **When to use:** When generating static or semi-static voxel meshes where reducing draw calls is critical and the specific culling rules (like bottom face culling) are acceptable.
*   **When not to use:** When dynamic voxel changes require frequent mesh updates, or when bottom faces need to be visible.

## Background
Face Culling is a common optimization in voxel terrain generation. The specific implementation described here, including the `Cube` and `Face` struct layouts and the unique side-face culling logic, originates from the `Voxel-Terrain-Generation` project by cadenmbond [S1, unofficial]. The Binary Hidden-Face Culling variant was described in a devlog for the game "YNL - Vozel" by Yunasawa [S2, unofficial].

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
