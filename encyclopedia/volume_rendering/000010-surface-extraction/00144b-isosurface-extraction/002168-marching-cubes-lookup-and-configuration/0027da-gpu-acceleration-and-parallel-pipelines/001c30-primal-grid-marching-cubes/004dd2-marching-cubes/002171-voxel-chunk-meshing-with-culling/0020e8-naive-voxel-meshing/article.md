# Naive Voxel Meshing

Also called: Classic Voxel Meshing, Face-by-face meshing, Naive Meshing, Naïve Voxel Mesh Generation

## Overview

Naive Voxel Meshing is a surface extraction algorithm that generates a mesh by iterating over every voxel in a chunk and emitting geometry for all six faces of every solid voxel, regardless of adjacency [S2, S5, unofficial]. It solves the problem of converting discrete voxel data into a format renderable by standard graphics pipelines without performing any face culling or vertex sharing [S1, S5, unofficial]. Unlike its sibling [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md), which checks neighbors to omit internal faces, Naive Voxel Meshing treats each voxel as an independent cube, resulting in a significantly higher polygon count and draw call overhead [S1, S2, unofficial].

## How it works

**Inputs and Outputs:**
*   **Input:** A voxel chunk defined by dimensions $W \times H \times D$ and a voxel access function `get_voxel(x, y, z)` returning a material ID (0 for empty/air, non-zero for solid).
*   **Output:** A list of `Quad` objects, where each quad represents a single unit face of a voxel.

**Data Structures:**
The algorithm produces a list of quads. Each quad is defined by the following fields [S1, S12, unofficial]:
*   `axis`: The normal axis ($0=\text{X}$, $1=\text{Y}$, $2=\text{Z}$).
*   `dir`: The direction sign ($+1$ for positive axis face, $-1$ for negative axis face).
*   `slice`: The integer coordinate along the axis where the face lies.
*   `u, v`: The starting coordinates on the plane's local axes.
*   `w, h`: The width and height of the face (always $1$ for naive meshing).
*   `material`: The ID of the voxel material.

**Algorithm Steps:**
1.  Initialize an empty list `quads`.
2.  Iterate through every voxel coordinate $(x, y, z)$ within the chunk bounds $[0, W) \times [0, H) \times [0, D)$ [S2, S5, unofficial].
3.  Retrieve the material ID of the current voxel. If the material ID is $0$ (empty), skip to the next voxel [S2, S5, unofficial].
4.  For each of the 6 face directions (defined by axis and direction sign):
    a. Calculate the neighbor's coordinates $(nx, ny, nz)$ by adding the direction offset to $(x, y, z)$ [S11, unofficial].
    b. Determine the neighbor's state:
        i. If $(nx, ny, nz)$ is outside the chunk bounds, apply the Boundary Policy (return $0$ for "Air", non-zero for "Solid") [S6, unofficial].
        ii. If inside bounds, lookup the voxel state at $(nx, ny, nz)$ [S2, unofficial].
    c. **Naive Variant:** In the strict naive implementation, neighbor checks are often skipped entirely, or the algorithm simply emits all 6 faces for every solid voxel without checking if the neighbor is solid [S2, S5, unofficial]. However, the reference implementation provided below includes neighbor checks but does not cull based on solid neighbors in the same way culled meshing does; rather, it generates the face if the neighbor is empty. *Note: The term "Naive" in this context primarily refers to the lack of vertex sharing and the generation of redundant geometry for internal faces if neighbor checks are omitted or if the "naive" definition implies emitting all faces regardless.* The provided pseudocode follows the logic where it checks neighbors but generates the face if the neighbor is empty, which is technically the same logic as face culling but without the optimization of skipping the neighbor lookup for solid neighbors, or it implies that "Naive" here refers to the specific implementation style where every voxel is processed individually with full face generation logic [S2, S5, unofficial]. *Correction based on sources:* Source [S2, unofficial] explicitly states Naive emits all six faces for every solid voxel without checking neighbors. Source [S5, unofficial] states it generates six quads for the solid voxel. The pseudocode below reflects the "emit all 6 faces" behavior for solid voxels, which is the defining characteristic of "Naive" vs "Culled" [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def naive_voxel_meshing(chunk_width, chunk_height, chunk_depth, get_voxel):
    """
    Generates a mesh for a voxel chunk using naive meshing.
    Emits all 6 faces for every solid voxel, ignoring neighbors.
    
    Args:
        chunk_width, chunk_height, chunk_depth: Dimensions of the chunk.
        get_voxel(x, y, z): Function returning the material ID of a voxel.
                           Returns 0 for empty/air.
                           
    Returns:
        A list of Quad objects.
    """
    quads = []
    
    # Define the 6 face directions: (axis_index, direction_sign, neighbor_offset_axis, neighbor_offset_sign)
    # axis_index: 0=X, 1=Y, 2=Z
    # direction_sign: +1 for positive axis face, -1 for negative axis face
    faces = [
        (0, 1, 1, 0, 0),  # +X face
        (0, -1, -1, 0, 0), # -X face
        (1, 1, 0, 1, 0),  # +Y face
        (1, -1, 0, -1, 0), # -Y face
        (2, 1, 0, 0, 1),  # +Z face
        (2, -1, 0, 0, -1)  # -Z face
    ]
    
    for z in range(chunk_depth):
        for y in range(chunk_height):
            for x in range(chunk_width):
                # Step 3: Skip empty voxels
                current_material = get_voxel(x, y, z)
                if current_material == 0:
                    continue
                
                # Step 4: Naive approach - emit all 6 faces for solid voxels
                # No neighbor checks are performed to cull internal faces [S2, S5, unofficial]
                for axis, dir_sign, nx_off, ny_off, nz_off in faces:
                    # Determine slice coordinate and local u,v
                    # For axis 0 (X), slice is x (for -X) or x+1 (for +X)
                    # Local axes u, v are the other two dimensions
                    if axis == 0:
                        slice_coord = x if dir_sign == -1 else x + 1
                        u, v = y, z
                    elif axis == 1:
                        slice_coord = y if dir_sign == -1 else y + 1
                        u, v = z, x
                    else: # axis == 2
                        slice_coord = z if dir_sign == -1 else z + 1
                        u, v = x, y
                        
                    quads.append({
                        "axis": axis,
                        "dir": dir_sign,
                        "slice": slice_coord,
                        "u": u,
                        "v": v,
                        "w": 1,
                        "h": 1,
                        "material": current_material
                    })
                        
    return quads
```

**Edge Cases and Pitfalls:**
*   **Precision:** Integer coordinates are used for slice and position, avoiding floating-point errors in face placement [S1, unofficial].
*   **Boundaries:** Since naive meshing does not check neighbors, boundary conditions are handled implicitly by the chunk iteration bounds. However, if the mesh is rendered as part of a larger world, the lack of culling at chunk edges may result in double-sided geometry if adjacent chunks also emit their outer faces [S1, unofficial].
*   **Translucency:** The basic algorithm assumes opaque voxels. If translucent blocks are present, the culling logic must be extended, but naive meshing typically ignores this, emitting all faces regardless of material properties [S7, S10, unofficial].

## Subtopics

### [Naive Surface Net](3679-naive-surface-net.md)
A GPU-accelerated variant that averages intersection points on voxel edges to create smoothed feature points before constructing quads, producing a smoother surface than standard voxel extraction.

### [Unindexed Triangle Meshing](10999-unindexed-triangle-meshing.md)
A technique that generates a complete, watertight cube geometry (12 triangles) for every active voxel, ignoring neighbor relationships and performing no face culling, explicitly decomposing the cube rather than treating it as a single primitive.

## Operating characteristics

*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the chunk. Each voxel is processed exactly once, and a constant number of operations (6 face checks/emissions) are performed per voxel.
*   **Memory Complexity:** $O(F)$, where $F$ is the number of faces generated. In the worst case (all voxels solid), $F = 6N$.
*   **Measured Performance:** On an Apple M2 Pro (macOS 26.5.1, Node v22.22.2, three@0.185.1), naive meshing of a 32³ terrain chunk with 17,191 solid voxels has a median time of 5.60 ms (range 5.12–6.30 ms, 21% spread) [S2, unofficial].
*   **Output Size:** On a solid 32³ cube, naive meshing produces 196,608 quads [S2, unofficial]. On a 32³ terrain chunk with 17,191 solid voxels, it produces 103,146 quads, 412,584 vertices, and 206,292 triangles [S2, unofficial].
*   **Strengths:** Simple implementation; no neighbor lookups required (if strictly naive); deterministic output.
*   **Limitations:** High poly counts; high number of draw calls due to lack of vertex sharing and face culling; inefficient for dense scenes [S1, S2, unofficial].
*   **When to use:** Prototyping; scenes with very low voxel density; when vertex sharing is handled by a separate post-processing step.
*   **When not to use:** Large, dense voxel worlds; performance-critical applications; scenes requiring efficient rendering of large solid volumes.

## Background

Naive Voxel Meshing is a foundational technique in voxel rendering, often cited as the baseline for comparison against more optimized methods like Face Culling and Greedy Meshing [S1, S2, unofficial]. It is described as "naive" because it does not use vertex sharing or advanced optimization in the core generation loop, treating each voxel as an independent entity [S4, S5, unofficial]. The approach is commonly associated with early voxel game engines and educational demonstrations, such as mrdoob's voxel demo [S1, unofficial].

## Sources

[S1] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S2] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S3] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S4] MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, https://github.com/MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, fetched 2026-10-27 (unofficial)
[S5] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
