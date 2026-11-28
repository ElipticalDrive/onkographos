# Naive Voxel Meshing

Also called: Classic Voxel Meshing, Face-by-face meshing, Naive Meshing, Naïve Voxel Mesh Generation, Per-voxel face culling

## Overview

Naive Voxel Meshing is a surface extraction algorithm that converts a 3D voxel grid into a renderable mesh by iterating over every voxel and emitting geometry for all six faces of every solid voxel, regardless of adjacency [S2, S5, unofficial]. It solves the problem of translating discrete voxel data into vertex/index buffers without performing face culling or vertex sharing [S1, S5, unofficial]. Unlike its sibling [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md), which checks neighbors to omit internal faces, Naive Voxel Meshing treats each voxel as an independent cube, resulting in a significantly higher polygon count and draw call overhead [S1, S2, unofficial].

## How it works

**Inputs and Outputs:**
*   **Input:** A voxel chunk defined by dimensions $W \times H \times D$ and a voxel access function `get_voxel(x, y, z)` returning a material ID (0 for empty/air, non-zero for solid) [S2, S5, unofficial].
*   **Output:** A list of `Quad` objects representing the exposed unit faces.

**Data Structures:**
Each generated face is represented as a `Quad` struct with the following fields [G]:
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
    a. Calculate the slice coordinate: if `dir` is $-1$, `slice` equals the voxel's coordinate along that axis; if `dir` is $+1$, `slice` equals the coordinate plus $1$ [S2, S6, unofficial].
    b. Determine the local plane axes $(u, v)$ as the two dimensions orthogonal to the face normal.
    c. Append a `Quad` to `quads` with the computed `axis`, `dir`, `slice`, `u`, `v`, `w=1`, `h=1`, and `material`.
5.  Return the `quads` list.

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
    
    # 6 face directions: (axis_index, direction_sign)
    # axis_index: 0=X, 1=Y, 2=Z
    # direction_sign: -1 (negative face), 1 (positive face)
    faces = [(0, -1), (0, 1), (1, -1), (1, 1), (2, -1), (2, 1)]
    
    for z in range(chunk_depth):
        for y in range(chunk_height):
            for x in range(chunk_width):
                # Step 3: Skip empty voxels
                current_material = get_voxel(x, y, z)
                if current_material == 0:
                    continue
                
                # Step 4: Naive approach - emit all 6 faces for solid voxels
                for axis, dir_sign in faces:
                    # Step 4a: Calculate slice coordinate
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

**Parameters:**
*   `chunk_width`, `chunk_height`, `chunk_depth`: Integer dimensions of the voxel chunk. Typical values are $16^3$ or $32^3$ [S2, S5, unofficial].
*   `get_voxel`: Callable returning material IDs. $0$ denotes empty space [S2, S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Integer coordinates are used for slice and position, avoiding floating-point errors in face placement [G].
*   **Boundaries:** Since naive meshing does not check neighbors, boundary conditions are handled implicitly by the chunk iteration bounds. All outer faces are always emitted, which may cause double-sided geometry or seams if adjacent chunks are rendered independently [G].
*   **Translucency:** The basic algorithm assumes opaque voxels. If translucent blocks are present, the logic must be extended to handle material transparency, but naive meshing typically ignores this, emitting all faces regardless of material properties [G].

## Subtopics

### [Naive Surface Net](3679-naive-surface-net.md)
A GPU-accelerated variant that averages intersection points on voxel edges to create smoothed feature points before constructing quads, producing a smoother surface than standard voxel extraction.

### [Unindexed Triangle Meshing](10999-unindexed-triangle-meshing.md)
A technique that generates a complete, watertight cube geometry (12 triangles) for every active voxel, ignoring neighbor relationships and performing no face culling, explicitly decomposing the cube rather than treating it as a single primitive.

## Operating characteristics

*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the chunk. Each voxel is processed exactly once, and a constant number of operations (6 face emissions) are performed per voxel [G].
*   **Memory Complexity:** $O(N)$, where $N$ is the number of voxels. In the worst case (all voxels solid), the output contains $6N$ quads [G].
*   **Measured Performance:** On an Apple M2 Pro (macOS 26.5.1, Node v22.22.2, three@0.185.1), naive meshing of a 32³ terrain chunk with 17,191 solid voxels has a median time of 5.60 ms (range 5.12–6.30 ms, 21% spread) [S2, unofficial].
*   **Output Size:** On a solid 32³ cube, naive meshing produces 196,608 quads [S2, unofficial]. On a 32³ terrain chunk with 17,191 solid voxels, it produces 103,146 quads, 412,584 vertices, and 206,292 triangles [S2, unofficial].
*   **Strengths:** Simple implementation; no neighbor lookups required; deterministic output; fast for prototyping [S1, S2, unofficial].
*   **Limitations:** High poly counts; high number of draw calls due to lack of vertex sharing and face culling; inefficient for dense scenes [S1, S2, unofficial].
*   **When to use:** Prototyping; scenes with very low voxel density; when vertex sharing is handled by a separate post-processing step [G].
*   **When not to use:** Large, dense voxel worlds; performance-critical applications; scenes requiring efficient rendering of large solid volumes [G].

## Background

Naive Voxel Meshing is a foundational technique in voxel rendering, often cited as the baseline for comparison against more optimized methods like Face Culling and Greedy Meshing [S1, S2, unofficial]. It is described as "naive" because it does not use vertex sharing or advanced optimization in the core generation loop, treating each voxel as an independent entity [S4, S5, unofficial]. The approach is commonly associated with early voxel game engines and educational demonstrations, such as mrdoob's voxel demo [S1, unofficial].

## Sources

[S1] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S2] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S3] jshrake/lexov, https://github.com/jshrake/lexov, fetched 2026-10-27 (unofficial)
[S4] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S5] MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, https://github.com/MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, fetched 2026-10-27 (unofficial)
[S6] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
