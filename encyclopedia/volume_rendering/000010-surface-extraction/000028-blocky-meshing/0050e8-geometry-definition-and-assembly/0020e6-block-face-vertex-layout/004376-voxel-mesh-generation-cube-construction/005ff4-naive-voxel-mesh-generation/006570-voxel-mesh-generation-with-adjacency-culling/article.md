# Voxel Mesh Generation with Adjacency Culling

Also called: Adjacency Culling Voxel Meshing, Face Culling Voxel Mesh Generation

## Overview
Voxel Mesh Generation with Adjacency Culling constructs a mesh from a voxel grid by iterating through active blocks and generating geometry only for faces adjacent to inactive blocks or world boundaries [S1, unofficial]. It prevents triangles from connecting across voxel boundaries, which is the primary failure mode of naive block-based meshing [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid `blocks[x][y][z]` where each entry is a boolean indicating block activity, and a `blockSize` scalar [S1, unofficial].
- Output: Two flat arrays, `WorldVertices` (floats) and `WorldIndices` (integers), representing the final mesh [S1, unofficial].

**Data Structures:**
- `WorldVertices`: A flat array of floats. Each vertex is represented by 3 consecutive floats $(x, y, z)$ [S1, unofficial].
- `WorldIndices`: A flat array of integers. Each triangle is represented by 3 consecutive indices into `WorldVertices` [S1, unofficial].

**Parameters:**
- `blockSize`: The scaling factor applied to voxel coordinates to determine world position [S1, unofficial]. The reference implementation uses `blockSize = 2` [S1, unofficial].

**Algorithm Steps:**
1. Initialize empty arrays `WorldVertices` and `WorldIndices` [S1, unofficial].
2. Iterate through the voxel grid using nested loops for $x$, $y$, and $z$ [S1, unofficial].
3. For each coordinate $(x, y, z)$, retrieve the block. If `block.isActive` is false, skip to the next iteration [S1, unofficial].
4. Calculate the world center of the block: $cx = x \cdot \text{blockSize}$, $cy = y \cdot \text{blockSize}$, $cz = z \cdot \text{blockSize}$ [S1, unofficial].
5. Define the 8 corner points of the cube relative to the center. Let $h = \text{blockSize} / 2$. The points are:
   - $p_0 = (cx - h, cy - h, cz + h)$
   - $p_1 = (cx + h, cy - h, cz + h)$
   - $p_2 = (cx + h, cy + h, cz + h)$
   - $p_3 = (cx - h, cy + h, cz + h)$
   - $p_4 = (cx + h, cy - h, cz - h)$
   - $p_5 = (cx - h, cy - h, cz - h)$
   - $p_6 = (cx - h, cy + h, cz - h)$
   - $p_7 = (cx + h, cy + h, cz - h)$ [S1, unofficial]
6. For each of the 6 faces, check the neighbor block in the outward direction. If the neighbor is inactive or out of bounds, generate the face geometry [S1, unofficial].
   - **Front Face** ($z+$): Neighbor at $(x, y, z+1)$. Vertices $p_0, p_1, p_2, p_3$.
   - **Back Face** ($z-$): Neighbor at $(x, y, z-1)$. Vertices $p_4, p_5, p_6, p_7$.
   - **Right Face** ($x+$): Neighbor at $(x+1, y, z)$. Vertices $p_1, p_4, p_7, p_2$.
   - **Left Face** ($x-$): Neighbor at $(x-1, y, z)$. Vertices $p_5, p_0, p_3, p_6$.
   - **Top Face** ($y+$): Neighbor at $(x, y+1, z)$. Vertices $p_3, p_2, p_7, p_6$.
   - **Bottom Face** ($y-$): Neighbor at $(x, y-1, z)$. Vertices $p_5, p_4, p_1, p_0$ [S1, unofficial].
7. For each visible face, add the 4 corner vertices to `WorldVertices` and record their indices. Then add the 2 triangles for that face to `WorldIndices` [S1, unofficial].
   - Front: $(p_0, p_1, p_2), (p_0, p_2, p_3)$
   - Back: $(p_4, p_5, p_6), (p_4, p_6, p_7)$
   - Right: $(p_1, p_4, p_7), (p_1, p_7, p_2)$
   - Left: $(p_5, p_0, p_3), (p_5, p_3, p_6)$
   - Top: $(p_3, p_2, p_7), (p_3, p_7, p_6)$
   - Bottom: $(p_5, p_4, p_1), (p_5, p_1, p_0)$ [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mesh(blocks, world_size, block_size):
    """
    blocks: 3D list/array of booleans, blocks[x][y][z] is True if active.
    world_size: tuple (max_x, max_y, max_z) defining the grid dimensions.
    block_size: float, size of one voxel in world units.
    """
    half_size = block_size / 2.0
    world_vertices = []
    world_indices = []

    def add_vertex(v):
        world_vertices.extend(v)
        return len(world_vertices) // 3 - 1

    def add_triangle(v0, v1, v2):
        world_indices.extend([v0, v1, v2])

    def is_neighbor_active(x, y, z):
        if 0 <= x < world_size[0] and 0 <= y < world_size[1] and 0 <= z < world_size[2]:
            return blocks[x][y][z]
        return False

    for x in range(world_size[0]):
        for y in range(world_size[1]):
            for z in range(world_size[2]):
                if not blocks[x][y][z]:
                    continue

                cx = x * block_size
                cy = y * block_size
                cz = z * block_size

                # Precompute corner vertices relative to world center
                p0 = [cx - half_size, cy - half_size, cz + half_size]
                p1 = [cx + half_size, cy - half_size, cz + half_size]
                p2 = [cx + half_size, cy + half_size, cz + half_size]
                p3 = [cx - half_size, cy + half_size, cz + half_size]
                p4 = [cx + half_size, cy - half_size, cz - half_size]
                p5 = [cx - half_size, cy - half_size, cz - half_size]
                p6 = [cx - half_size, cy + half_size, cz - half_size]
                p7 = [cx + half_size, cy + half_size, cz - half_size]

                # Front Face (z+)
                if not is_neighbor_active(x, y, z + 1):
                    v0, v1, v2, v3 = add_vertex(p0), add_vertex(p1), add_vertex(p2), add_vertex(p3)
                    add_triangle(v0, v1, v2)
                    add_triangle(v0, v2, v3)

                # Back Face (z-)
                if not is_neighbor_active(x, y, z - 1):
                    v4, v5, v6, v7 = add_vertex(p4), add_vertex(p5), add_vertex(p6), add_vertex(p7)
                    add_triangle(v4, v5, v6)
                    add_triangle(v4, v6, v7)

                # Right Face (x+)
                if not is_neighbor_active(x + 1, y, z):
                    v1, v4, v7, v2 = add_vertex(p1), add_vertex(p4), add_vertex(p7), add_vertex(p2)
                    add_triangle(v1, v4, v7)
                    add_triangle(v1, v7, v2)

                # Left Face (x-)
                if not is_neighbor_active(x - 1, y, z):
                    v5, v0, v3, v6 = add_vertex(p5), add_vertex(p0), add_vertex(p3), add_vertex(p6)
                    add_triangle(v5, v0, v3)
                    add_triangle(v5, v3, v6)

                # Top Face (y+)
                if not is_neighbor_active(x, y + 1, z):
                    v3, v2, v7, v6 = add_vertex(p3), add_vertex(p2), add_vertex(p7), add_vertex(p6)
                    add_triangle(v3, v2, v7)
                    add_triangle(v3, v7, v6)

                # Bottom Face (y-)
                if not is_neighbor_active(x, y - 1, z):
                    v5, v4, v1, v0 = add_vertex(p5), add_vertex(p4), add_vertex(p1), add_vertex(p0)
                    add_triangle(v5, v4, v1)
                    add_triangle(v5, v1, v0)

    return world_vertices, world_indices
```

**Edge Cases and Pitfalls:**
- **Boundary Checks:** The algorithm must correctly handle neighbors that are out of the grid bounds. Out-of-bounds neighbors are treated as inactive, ensuring the outer surface of the world is rendered [S1, unofficial].
- **Incorrect Neighbor Check:** A common pitfall is generating all 6 faces for every active block without checking neighbors. This results in internal faces being drawn, causing visual artifacts where triangles connect across voxels [S1, unofficial].
- **Index Offsets:** When using a template-based approach, failing to update index offsets correctly can cause skewed triangles. The adjacency culling approach above uses dynamic vertex IDs, avoiding this issue [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active blocks. Each active block processes a constant number of faces (up to 6) [G].
- **Memory Complexity:** $O(F)$, where $F$ is the number of exposed faces. The vertex and index arrays grow proportionally to the surface area of the active voxels [G].
- **Performance:** Significantly reduces the number of triangles compared to naive generation, especially for solid objects. For a solid $N \times N \times N$ cube, naive generation creates $6N^3$ triangles, while adjacency culling creates $6N^2$ triangles [G].
- **Strengths:** Simple to implement; effectively removes internal geometry; reduces draw calls if combined with batching [G].
- **Limitations:** Does not merge vertices or optimize topology (e.g., quad fans); still generates duplicate vertices for shared edges [G].

## Background
This method is a direct correction to naive voxel mesh generation, addressing the specific issue of internal face rendering. It was discussed in the context of WebGL voxel engine development to fix skewed triangles and hidden geometry artifacts [S1, unofficial].

## Sources
[S1] How to keep triangles from connecting across voxel mesh, https://gamedev.stackexchange.com/questions/134505/how-to-keep-triangles-from-connecting-across-voxel-mesh, fetched 2026-10-26 (unofficial)
