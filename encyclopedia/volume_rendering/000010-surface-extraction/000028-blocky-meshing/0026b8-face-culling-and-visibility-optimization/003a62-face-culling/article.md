# Face Culling

Also called: Exposure Culling, Neighbor Check Culling, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors [S1, unofficial]. It sets itself apart from siblings like [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by focusing on immediate neighbor checks during the initial mesh construction phase rather than block-level placement logic or coverage masks.

## How it works
**Inputs and Outputs:**
- Input: A set of occupied voxel positions in a uniform grid.
- Output: A mesh buffer containing only exposed faces.

**Data Structures:**
The implementation relies on specific structures to track voxel state and face properties [S1, unofficial]:
- `Cube` struct:
  - `std::vector<Face> faces`: A list of 6 faces, one for each direction ($\pm x, \pm y, \pm z$).
  - `glm::ivec3 cubeCenter`: The integer coordinates of the voxel center.
  - `glm::ivec3 gridPos`: The position in the global grid.
  - `int neighbors`: A counter for adjacent occupied voxels.
- `Face` struct:
  - `std::vector<float> vertices`: The vertex data for the face.
  - `glm::ivec3 normal`: The direction vector of the face normal.
  - `bool isSideFace`: Indicates if the face is a side face (not top/bottom).
  - `bool isBottomFace`: Indicates if the face is the bottom face.
  - `bool renderFlag`: A boolean flag determining if the face should be rendered.

**Algorithm Steps:**
1. Iterate through each occupied voxel (Cube) in the grid.
2. For each of the 6 faces of the voxel:
   a. Calculate the position of the immediate neighbor in the direction of the face normal:
      $$ \text{neighborPos} = \text{cubeCenter} + \text{face.normal} $$
   b. Check if a voxel exists at `neighborPos` in the `occupiedPositions` set.
   c. Determine the `renderFlag` based on the face type and neighbor existence:
      - If the face is a **bottom face** (`isBottomFace` is true):
        - Set `renderFlag = false`. The bottom face is always culled regardless of neighbors below [S1, unofficial].
      - If the face is a **side face** (`isBottomFace` is false):
        - Check if a neighbor exists directly above the neighbor position:
          $$ \text{aboveNeighborPos} = \text{neighborPos} + (0, 1, 0) $$
        - If a voxel exists at `aboveNeighborPos`, set `renderFlag = false`. This culls side faces if they are occluded by a voxel directly above the adjacent voxel [S1, unofficial].
        - Otherwise, if a voxel exists at `neighborPos`, set `renderFlag = false` (internal face).
        - If no voxel exists at `neighborPos` or `aboveNeighborPos`, set `renderFlag = true` (exposed face) [S1, unofficial].
3. Add the face to the mesh buffer if `renderFlag` is true.
4. (Optional) Count neighbors for statistics or further optimization by checking specific offsets (e.g., the 6 offsets listed in the source code for `aboveOffsets`) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(occupied_positions: set[tuple[int, int, int]]) -> list[Face]:
    """
    Generates a mesh by culling faces adjacent to occupied neighbors.
    Assumes occupied_positions contains (x, y, z) tuples for all solid voxels.
    """
    mesh_faces = []
    
    # Define face normals for the 6 directions
    # Order: +x, -x, +y, -y, +z, -z
    # Note: The source code implies specific handling for 'bottom' vs 'side'.
    # We assume +y is Up, so -y is Bottom.
    face_normals = [
        (1, 0, 0),   # Right
        (-1, 0, 0),  # Left
        (0, 1, 0),   # Top
        (0, -1, 0),  # Bottom
        (0, 0, 1),   # Front
        (0, 0, -1)   # Back
    ]
    
    for cube_center in occupied_positions:
        cx, cy, cz = cube_center
        
        for i, normal in enumerate(face_normals):
            nx, ny, nz = normal
            
            # Calculate neighbor position
            neighbor_pos = (cx + nx, cy + ny, cz + nz)
            
            # Determine if this is a bottom face
            # Based on source: isBottomFace corresponds to normal (0, -1, 0)
            is_bottom_face = (nx == 0 and ny == -1 and nz == 0)
            
            render_flag = False
            
            if is_bottom_face:
                # Bottom faces are always culled
                render_flag = False
            else:
                # Side faces (including Top)
                # Check if neighbor exists directly adjacent
                neighbor_exists = neighbor_pos in occupied_positions
                
                # Check if neighbor exists directly above the neighbor position
                # Source code: neighborPos + glm::ivec3(0, 1, 0)
                above_neighbor_pos = (neighbor_pos[0], neighbor_pos[1] + 1, neighbor_pos[2])
                above_neighbor_exists = above_neighbor_pos in occupied_positions
                
                if above_neighbor_exists:
                    render_flag = False
                elif neighbor_exists:
                    render_flag = False
                else:
                    render_flag = True
            
            if render_flag:
                # Create face geometry here
                # face = create_face_geometry(cube_center, normal)
                # mesh_faces.append(face)
                pass # Placeholder for geometry creation logic not detailed in source
                
    return mesh_faces
```

**Parameters:**
- Grid resolution: Not specified, but assumes integer coordinates [S1, unofficial].
- Performance: 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Bottom Face Culling:** The algorithm explicitly culls all bottom faces (`renderFlag = false`) regardless of whether there is a voxel below it [S1, unofficial]. This may be intentional for terrain generation (hiding the underside) but results in non-manifold geometry if the bottom is exposed to air and needs to be rendered.
- **Side Face Occlusion Logic:** Side faces are culled not only if the immediate neighbor exists, but also if a neighbor exists directly *above* the neighbor position. This implies a specific occlusion model where side faces are hidden by overhangs or adjacent blocks at a higher level [S1, unofficial].
- **Precision:** The method assumes a uniform grid with integer coordinates. Floating-point voxel centers would require rounding or grid quantization before neighbor lookup.

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It sets itself apart from standard neighbor checks by operating on entire rows or columns of voxels simultaneously using bitwise shifts, avoiding per-voxel loops and random memory access [S2, unofficial]. Choose this method when performance is critical and voxels can be efficiently mapped to linear bitmasks (e.g., single-axis slices).
- **Definition:** Uses bitmasks where 1 indicates a solid voxel and 0 indicates air [S2, unofficial].
- **Steps:**
  1. Represent a slice of voxels as a bitmask $m$.
  2. To find visible left faces: Shift $m$ right by 1 ($m \gg 1$), invert the result ($\sim(m \gg 1)$), and AND with original mask: $m \ \& \ \sim(m \gg 1)$ [S2, unofficial].
  3. To find visible right faces: Shift $m$ left by 1 ($m \ll 1$), invert, and AND: $m \ \& \ \sim(m \ll 1)$ [S2, unofficial].
  4. Use the resulting masks to determine which faces to draw [S2, unofficial].
- **Pitfall:** The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V)$ where $V$ is the number of voxels, assuming $O(1)$ lookup in the `occupiedPositions` set (hash set) or array [S1, unofficial]. The binary variant reduces this to $O(V/w)$ where $w$ is the word size, due to bitwise parallelism [S2, unofficial].
- **Memory Complexity:** $O(V)$ to store the occupied positions and the resulting mesh. The binary variant requires $O(V/w)$ bits for the mask [S2, unofficial].
- **Measured Performance:** 4096 cubes rendered at ~60 FPS on the author's system [S1, unofficial].
- **Strengths:** Significantly reduces vertex count sent to the GPU by eliminating internal faces. The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S1, unofficial].
- **Limitations:** The specific pitfall of always culling bottom faces may require post-processing or separate handling if the bottom surface needs to be visible. The side-face occlusion logic (checking above neighbors) is specific to the source implementation and may not generalize to all voxel rendering needs without modification [S1, unofficial].
- **When to use:** When generating static voxel meshes where internal faces are not needed and performance of mesh generation is a bottleneck.
- **When not to use:** When dynamic voxel changes require frequent re-culling, or when the bottom surface must be rendered.

## Background
Face Culling in voxel meshing originated as a basic optimization to avoid rendering hidden geometry. The specific implementation detailed in [S1, unofficial] is from a GitHub repository for voxel terrain generation. The binary variant [S2, unofficial] was developed as an optimization for a Minecraft-inspired project (YNL - Vozel) to address performance issues with random memory access in neighbor checks.

## Sources
[S1] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
