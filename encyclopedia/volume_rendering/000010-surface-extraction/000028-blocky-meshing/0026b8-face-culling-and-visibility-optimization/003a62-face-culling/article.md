# Face Culling

Also called: Exposure Culling, Face Occlusion Culling, Neighbor Check Culling, Only render block faces that aren't adjacent to another face, Voxel Neighbor Face Culling

## Overview
Face Culling is a mesh generation optimization that prevents the rendering of block faces occluded by adjacent solid voxels, significantly reducing triangle count in homogeneous voxel terrain [S1, S2, unofficial]. It solves the problem of naively rendering all six faces of every cube, including those hidden by neighbors, by skipping the generation of faces where the adjacent voxel coordinate is also solid [S2, unofficial]. Unlike [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) which operates on the GPU or during rasterization, this method performs static occlusion detection during mesh creation to minimize vertex buffer size and draw calls [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid (or chunk) representing occupancy states (solid/air) and a list of candidate voxel positions.
*   **Output:** A vertex buffer containing only the exposed faces of solid voxels.

**Data Structures:**
The implementation typically maintains a fast lookup for occupied positions. A common structure involves a `Cube` object and a global set of occupied coordinates [S2, unofficial]:
*   `occupiedPositions`: A hash set (`std::unordered_set<glm::ivec3>`) or boolean array mapping 3D integer coordinates to occupancy status.
*   `Cube` struct:
    *   `cubeCenter`: `glm::ivec3` representing the voxel's position in the world grid.
    *   `faces`: An array or vector of 6 `Face` objects, one for each axis-aligned direction.
    *   `neighbors`: An integer counter for adjacent solid voxels.
*   `Face` struct:
    *   `vertices`: A vector of floats representing the geometric vertices of the face.
    *   `normal`: `glm::ivec3` indicating the face's outward direction.
    *   `isSideFace`: Boolean flag distinguishing side faces from bottom/top faces.
    *   `isBottomFace`: Boolean flag indicating if the face is the bottom of the voxel.
    *   `renderFlag`: Boolean flag determining if the face is added to the final mesh.

**Algorithm Steps:**
1.  Iterate through every solid voxel in the chunk or world.
2.  For each of the six faces of the voxel, determine the coordinate of the adjacent neighbor: $P_{neighbor} = P + N$, where $P$ is the current voxel position and $N$ is the face normal vector [S1, S2, unofficial].
3.  Check if $P_{neighbor}$ exists in the `occupiedPositions` set.
4.  Apply culling logic based on face type and neighbor existence:
    *   **Bottom Faces:** Always culled (set `renderFlag` to false), regardless of the neighbor below [S2, unofficial].
    *   **Side/Top Faces:**
        *   If a solid neighbor exists at $P_{neighbor}$, the face is internal and culled (`renderFlag` = false) [S1, S2, unofficial].
        *   If no solid neighbor exists at $P_{neighbor}$, the face is exposed and rendered (`renderFlag` = true) [S2, unofficial].
        *   *Note:* Some implementations include additional checks for "overhang" or specific side-face occlusion logic where a neighbor directly above the neighbor position ($P_{neighbor} + (0,1,0)$) might influence the flag, though the primary definition relies on direct adjacency [S2, unofficial].
5.  If `renderFlag` is true, append the face's vertices to the vertex buffer.
6.  The culling check runs once during mesh creation, resulting in no framerate loss during rendering [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(voxel_grid, chunk_bounds):
    """
    Generates a mesh for a chunk by culling internal faces.
    voxel_grid: A 3D array or hash map where voxel_grid[x,y,z] is True if solid.
    chunk_bounds: The min and max coordinates of the chunk.
    """
    vertices = []
    
    # Define face normals and vertex offsets for a unit cube
    # Normals: +X, -X, +Y, -Y, +Z, -Z
    normals = [
        (1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)
    ]
    
    # Vertex definitions for each face (simplified for 2 triangles per face)
    # In practice, these are generated based on the normal and position
    face_vertices = get_face_vertices() 

    for x, y, z in chunk_bounds:
        if not voxel_grid[x, y, z]:
            continue
            
        for i, (nx, ny, nz) in enumerate(normals):
            # Calculate neighbor position
            neighbor_x = x + nx
            neighbor_y = y + ny
            neighbor_z = z + nz
            
            # Check if neighbor is solid
            # Note: Boundary checks for chunk edges are critical here
            is_neighbor_solid = voxel_grid.get(neighbor_x, neighbor_y, neighbor_z, default=False)
            
            # Culling Logic
            render_face = True
            
            # 1. Bottom face culling (Y- direction)
            if ny == -1:
                render_face = False
                
            # 2. Neighbor occlusion culling
            elif is_neighbor_solid:
                render_face = False
                
            # 3. Optional: Specific side-face overhang checks (from [S2, unofficial])
            # If the source code implies checking above the neighbor for side faces:
            # if ny == 0 and voxel_grid.get(x, y+1, z, default=False):
            #     render_face = False 
            # However, standard neighbor check [S1, unofficial] is sufficient for basic culling.
            
            if render_face:
                # Generate vertices for this face at (x,y,z) with normal (nx,ny,nz)
                face_verts = generate_face_geometry(x, y, z, nx, ny, nz)
                vertices.extend(face_verts)
                
    return vertices

def get_face_vertices():
    # Placeholder for actual vertex generation logic
    # Returns 4 vertices (2 triangles) for a unit face
    pass

def generate_face_geometry(x, y, z, nx, ny, nz):
    # Generates 4 vertices for the face at x,y,z facing nx,ny,nz
    # This depends on the specific vertex layout (e.g., quad vs triangles)
    # [G] Standard implementation creates a quad centered on the face
    return [] 
```

**Parameters:**
*   **Chunk Size:** Commonly $16 \times 16 \times 16$ voxels [S1, unofficial].
*   **Neighbor Check Scope:** Must include adjacent chunks to prevent boundary artifacts [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Chunk Boundaries:** Adjacent cubes across chunk boundaries (e.g., $x=0$ and $x=15$) must be checked. If ignored, the outline of each chunk is drawn, killing performance [S1, unofficial].
*   **Bottom Face Culling:** Bottom faces are often always culled to reduce draw calls, assuming the ground is opaque or not visible from below [S2, unofficial].
*   **Precision:** Integer coordinate arithmetic is used; floating-point precision issues do not apply to the culling logic itself, only to the subsequent vertex generation [S2, unofficial].

## Subtopics

### Binary Hidden-Face Culling
A technique to reduce the number of faces in a voxel mesh by removing those completely hidden (adjacent to another voxel), using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S3, unofficial]. It sets itself apart by processing entire rows or columns of voxels simultaneously using shifts and bitwise AND/OR operations, avoiding per-voxel branching and random memory access [S3, unofficial]. Choose this method when generating meshes for long, linear structures (like tunnels or walls) where SIMD-friendly bitwise operations can outperform hash lookups. Pitfall: The original approach of checking each of the six faces by looping through neighbors and performing random memory access leads to very poor performance [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel and its 6 neighbors are checked once. The bitwise variant reduces constant factors significantly for linear data [S3, unofficial].
*   **Memory:** Proportional to the number of exposed faces. In homogeneous terrain, this reduces the vertex count by approximately 99.9% (drawing ~1/1000 of the triangles previously drawn) [S1, unofficial].

**Measured Performance:**
*   On an Intel Core i7 2.5 GHz and AMD Radeon HD 6770M 1024 MB, this optimization allows running more than 14,440 chunks while performing GPU-heavy algorithms like cascaded shadow mapping and water shading [S1, unofficial].
*   Without this optimization, performance degrades rapidly, with lag occurring at ~20 chunks [S1, unofficial].
*   Specific implementation tests show 4096 cubes rendered at ~60 FPS [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Drastic reduction in triangle count; no runtime overhead during rendering (culling happens at mesh build time); simple to implement [S1, S2, unofficial].
*   **Limitations:** Requires rebuilding the mesh if the voxel data changes; does not handle view-dependent culling (backface/frustum) which must be handled separately [S1, unofficial].

**When to Use:**
*   Use for static or semi-static voxel terrain where mesh updates are infrequent.
*   Do not use if the scene is highly dynamic with frequent voxel changes and low-latency mesh updates are required, as the rebuild cost may outweigh the rendering savings.

## Background
Face Culling originated as a fundamental optimization in voxel-based game development, particularly in the Minecraft-like genre. It was widely discussed in community forums around 2016 as a critical step for rendering large voxel worlds on consumer hardware [S1, unofficial]. The technique is foundational to modern voxel engines, often serving as the first optimization before more complex methods like Greedy Meshing or GPU-based culling are implemented [S1, unofficial].

## Sources
[S1] Optimising Voxel World Rendering, https://gamedev.stackexchange.com/questions/117398/optimising-voxel-world-rendering, fetched 2026-10-27 (unofficial)
[S2] cadenmbond/Voxel-Terrain-Generation, https://github.com/cadenmbond/Voxel-Terrain-Generation, fetched 2026-10-27 (unofficial)
[S3] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
