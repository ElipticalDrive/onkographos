# Greedy Meshing

Also called: GreedMesh, Greedy Mesh, Greedy Mesher, Greedy Quad Merging, Greedy Quad Meshing, Greedy Voxel Meshing, Greedy face merging, Maximal Rectangle Detection, Naive Greedy Meshing, Rectangle Merging, Voxel Face Crawling, Voxel Quad Merging

## Overview
Greedy meshing is a surface extraction algorithm that reduces the vertex and primitive count of a voxel mesh by merging adjacent, coplanar faces of identical material into maximal axis-aligned quads [S1, S3, S4, unofficial]. It solves the problem of excessive geometry in naive voxel rendering (where every exposed voxel face becomes a separate quad) by scanning the volume along each of the three principal axes and greedily expanding rectangles of identical visible faces [S1, S3, S7, unofficial]. Unlike [Isosurface extraction](30-isosurface-extraction.md), which interpolates scalar fields to create smooth surfaces, greedy meshing preserves the exact blocky aesthetic of the discrete grid [S3, S13, unofficial]. It differs from [Blocky meshing](31-blocky-meshing.md) siblings like [Cuberille Method](1718-cuberille-method.md) by actively merging adjacent primitives into larger rectangles rather than just culling internal faces [S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid $V$ of dimensions $D_x \times D_y \times D_z$. Each cell $V(x,y,z)$ contains a material ID (or occupancy state). A boundary policy (e.g., "air" or "solid") determines how out-of-bounds neighbors are treated [S12, S13, unofficial].
*   **Output:** A list of quads. Each quad is defined by its origin $(x,y,z)$, its normal direction (one of $\pm \hat{x}, \pm \hat{y}, \pm \hat{z}$), and its dimensions $(w, h)$ in voxel units [S20, S27, unofficial].

**Data Structures:**
*   **Mask:** A 2D boolean or integer array of size $D_u \times D_v$ (where $u, v$ are the two axes perpendicular to the current sweep normal) used to track which cells in the current slice have been processed or to store the visibility/material state [S1, S7, S27, unofficial].
*   **Visited/Processed:** A 3D boolean array or bitmask array of size $D_x \times D_y \times D_z$ to ensure each voxel face is merged only once, preventing double-counting [S22, S28, unofficial].

**Algorithm Steps:**
The algorithm iterates over the three principal axes (normals). For each normal direction $n \in \{0, 1, 2\}$ (representing X, Y, Z), it defines the tangent $t = (n + 1) \pmod 3$ and bitangent $b = (n + 2) \pmod 3$ [S1, S7, unofficial].

1.  **Sweep Slices:** Iterate through the volume along the normal axis $n$. For each slice index $s$ from $0$ to $D_n - 1$:
    *   Construct a 2D mask $M$ of size $D_t \times D_b$.
    *   For each cell $(t, b)$ in the slice at $s$:
        *   Determine if the face is exposed. A face is exposed if the voxel at $(x,y,z)$ (where $n$-coordinate is $s$) has a different material or is solid while the neighbor in the normal direction is empty (or vice versa, depending on culling logic). Specifically, compare $V(s, t, b)$ with $V(s - 1, t, b)$ if sweeping in the positive direction, or $V(s + 1, t, b)$ if sweeping in the negative direction [S7, S13, S19, unofficial].
        *   Set $M[t][b]$ to the material ID if exposed, or $0$ (or false) if not [S27, S31, unofficial].
    *   **Greedy Expansion:** Iterate through the mask $M$ in lexicographic order (row by row, then column by column) to find maximal rectangles:
        *   Find the first cell $(t_0, b_0)$ in $M$ that is non-zero (exposed) and not yet processed [S22, S27, unofficial].
        *   **Expand Width ($w$):** Increment $w$ starting at 1 while $t_0 + w < D_t$, the cell $(t_0 + w, b_0)$ is non-zero, and the material ID matches the starting cell's ID [S1, S27, unofficial].
        *   **Expand Height ($h$):** Increment $h$ starting at 1 while $b_0 + h < D_b$. For each potential height, verify that *all* cells in the rectangle defined by $[t_0, t_0 + w) \times [b_0, b_0 + h)$ have the same material ID and are non-zero [S1, S27, unofficial].
        *   **Emit Quad:** Create a quad with origin $(x,y,z)$ corresponding to $(t_0, b_0, s)$, normal $n$, width $w$, and height $h$ [S27, unofficial].
        *   **Mark Processed:** Mark all cells in the rectangle $[t_0, t_0 + w) \times [b_0, b_0 + h)$ as processed (e.g., set mask to 0 or update a visited array) to prevent them from being part of another quad in this pass [S21, S27, unofficial].
        *   Advance the scan cursor to $t_0 + w$ [S27, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh(volume, dims):
    """
    volume: function (x, y, z) -> int (material ID)
    dims: tuple (Dx, Dy, Dz)
    Returns: list of quads, each quad is (origin_xyz, normal_axis, width, height)
    """
    quads = []
    
    # Iterate over the 3 principal axes as normals
    for norm in range(3):
        tan = (norm + 1) % 3
        biTan = (norm + 2) % 3
        
        # Iterate through slices along the normal axis
        for slice_idx in range(dims[norm]):
            # Initialize 2D mask for this slice
            # Mask stores material ID if face is exposed, 0 otherwise
            mask = [[0 for _ in range(dims[biTan])] for _ in range(dims[tan])]
            
            # Build mask: check visibility against neighbor in normal direction
            # We check the face between slice_idx and slice_idx-1 (or +1 depending on convention)
            # Here we assume we are extracting faces facing the +normal direction
            # by comparing slice_idx with slice_idx-1.
            # Note: Boundary conditions (slice_idx=0) need handling (e.g., treat outside as air)
            
            for b in range(dims[biTan]):
                for t in range(dims[tan]):
                    # Construct coordinates
                    coords = [0, 0, 0]
                    coords[norm] = slice_idx
                    coords[tan] = t
                    coords[biTan] = b
                    
                    # Get current voxel
                    curr_voxel = volume(*coords)
                    
                    # Get neighbor in the -normal direction to determine if face is exposed
                    # If norm=0 (X), neighbor is at x-1. If x=0, neighbor is out of bounds.
                    neighbor_coords = coords[:]
                    neighbor_coords[norm] -= 1
                    
                    if neighbor_coords[norm] < 0:
                        neighbor_voxel = 0 # Treat out of bounds as empty/air
                    else:
                        neighbor_voxel = volume(*neighbor_coords)
                    
                    # Face is exposed if current is solid and neighbor is empty (or different material)
                    # Assuming 0 is empty/air
                    if curr_voxel != 0 and curr_voxel != neighbor_voxel:
                        mask[b][t] = curr_voxel
                    else:
                        mask[b][t] = 0

            # Greedy expansion on the 2D mask
            for b in range(dims[biTan]):
                t = 0
                while t < dims[tan]:
                    if mask[b][t] == 0:
                        t += 1
                        continue
                    
                    # Start of a new quad
                    start_material = mask[b][t]
                    start_coords = [0, 0, 0]
                    start_coords[norm] = slice_idx
                    start_coords[tan] = t
                    start_coords[biTan] = b
                    
                    # Expand Width (along tan axis)
                    width = 1
                    while t + width < dims[tan] and mask[b][t + width] == start_material:
                        width += 1
                    
                    # Expand Height (along biTan axis)
                    height = 1
                    while b + height < dims[biTan]:
                        # Check if the entire row segment of width matches
                        match = True
                        for k in range(width):
                            if mask[b + height][t + k] != start_material:
                                match = False
                                break
                        if not match:
                            break
                        height += 1
                    
                    # Emit Quad
                    quads.append((start_coords, norm, width, height))
                    
                    # Clear the processed area from the mask to avoid re-processing
                    for h in range(height):
                        for w in range(width):
                            mask[b + h][t + w] = 0
                    
                    # Advance t by width
                    t += width

    return quads
```

**Parameters:**
*   **Merge Key:** By default, quads merge if they have the same material ID. Variants allow a custom `mergeKey(material, x, y, z)` function to include attributes like ambient occlusion, UV offset, or transparency [S12, S24, S30, unofficial].
*   **Boundary Policy:** Determines how out-of-bounds voxels are treated during visibility checks (e.g., always air, always solid) [S12, S13, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Integer coordinates are sufficient for the mesh structure. Vertex positions are derived by scaling integer coordinates by `VOXEL_SIZE` [S6, S9, unofficial].
*   **Boundaries:** The mask construction must correctly handle voxels at the chunk boundaries. Neighbors outside the chunk must be queried from adjacent chunks or treated as air/solid based on the boundary policy [S13, S16, unofficial].
*   **Degenerate Input:** Empty volumes produce no quads. Fully solid volumes produce no exposed faces (depending on boundary policy).

## Subtopics

### [Voxel Mesh Generation (Run-based Greedy Meshing)](3127-voxel-mesh-generation-run-based-greedy-meshing.md)
A CPU-based variant that merges adjacent visible faces into runs along the X and Y axes. It produces approximately 20% more triangles than full greedy meshing but runs approximately 390% faster, making it suitable for real-time scenarios requiring frequent chunk regeneration.

### [Chunked Voxel Mesh Generation with Face Culling and Quad Expansion](3133-chunked-voxel-mesh-generation-with-face-culling-and-quad-exp.md)
Processes 3D voxel chunks to generate simplified triangle meshes by merging adjacent coplanar faces. It explicitly handles neighbor chunk references to determine visibility across chunk boundaries and uses a packed integer format for vertex attributes to minimize memory bandwidth. Unlike binary variants, it supports multiple voxel types and ambient occlusion.

### [Binary Greedy Meshing with Packed Quads](3312-binary-greedy-meshing-with-packed-quads.md)
Operates on packed binary planes (1 bit per voxel) using bitwise operations (trailing zeros, shifts, masks) to greedily expand rectangles. This reduces clock cycles significantly compared to branching logic used in standard greedy meshing. It focuses on binary occupancy and achieves up to 54.5x memory reduction compared to Marching Cubes.

### [Greedy Meshing for Voxel Faces](3367-greedy-meshing-for-voxel-faces.md)
Extends standard greedy meshing by merging faces based on multiple attributes (material type, transparency) using a custom equality check on face objects. This allows for complex culling and attribute-based optimization beyond simple voxel existence.

### [3D Greedy Meshing](3519-3d-greedy-meshing.md)
Decomposes the 3D problem into six directional 2D sub-problems by extracting 2D slices from the 3D voxel grid along each axis. It generates a reduced set of faces representing the external surface, achieving at most 8x the number of faces of a perfect mesh.

### [Greedy Meshing with Vertex Pooling](6808-greedy-meshing-with-vertex-pooling.md)
Applies greedy meshing by iterating over each surface orientation separately, assigning groups based on orientation, and utilizing a vertex pool bucket system to manage memory and draw calls.

### [Voxel Mesh Generation (Run-based Greedy Meshing)](3127-voxel-mesh-generation-run-based-greedy-meshing.md)
A specific implementation entry point that initializes output arrays for 6 face directions, calls the core meshing function (`mesh_face_columns_all_x_slices`), and calculates vertex counts per face. It caps the maximum number of face quads per chunk to `CS * CS * CS`.

### [Binary Greedy Meshing with Packed Quads](3312-binary-greedy-meshing-with-packed-quads.md)
Applies greedy meshing to 4D voxel worlds by sweeping in lattice space (4D) rather than slice space (3D). It achieves reduction ratios of 1.41x to 3.58x depending on cut configuration, with an overall 5.3x reduction compared to naive per-face meshing. Re-merging in a third dimension recovers only about 1% of quads.

### [Chunked Voxel Mesh Generation with Face Culling and Quad Expansion](3133-chunked-voxel-mesh-generation-with-face-culling-and-quad-exp.md)
Strategies to handle ambient occlusion (AO) variation during merging. Options include restricted merging (only merging if AO values are identical), texture atlas baking (pre-computed AO gradients), or fragment-shader reconstruction (analytical edge shading).

### [Voxel Mesh Generation (Run-based Greedy Meshing)](3127-voxel-mesh-generation-run-based-greedy-meshing.md)
An optimization applied after greedy meshing to reduce vertex count from 6 vertices per face (naive) to 4 vertices per face (shared vertices). This further reduces the quad count by 1/3 compared to the greedy meshing output alone.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is visited a constant number of times (once per axis slice). The greedy expansion step is linear in the size of the 2D mask [S1, S7, unofficial].
*   **Memory Complexity:** $O(N)$ for the input volume and $O(D_u \times D_v)$ for the temporary 2D mask per slice. The output size is proportional to the surface area of the structure [S1, S7, unofficial].
*   **Measured Performance:** Greedy meshing reduces the number of quads by an average of 60% compared to naive face culling, and further optimizations (like VBO indexing) can reduce vertex count by another 1/3 [S4, unofficial]. In real-time scenarios, run-based variants can be ~390% faster than full greedy meshing with only a 20% increase in triangle count [S1, unofficial].
*   **Strengths:** Significantly reduces draw calls and vertex count (10-20x reduction compared to naive per-face rendering) [S14, S27, unofficial]. Simple to implement on CPU [S1, S7, unofficial].
*   **Limitations:** Not optimal for dynamic terrain where frequent updates occur; the global greedy approach can be slow to re-mesh small changes. Variants like run-based meshing are preferred for dynamic updates [S1, unofficial]. Does not handle transparency optimally without keeping quads unit-sized [S30, unofficial].

## Background
Greedy meshing was popularized by Mikola Lysenko in his 2012 article "Meshing in a Minecraft Game" [S1, S5, unofficial][S26]. It is a technique to adapt voxels, which are hard to compute, into large quads, making them easier and in many cases possible at all to render efficiently [S11, unofficial]. The algorithm is often cited alongside "Stupid" and "Culled" meshing as one of three primary approaches to voxel meshing [S1, unofficial].

## Sources
[S1] Are there other/better voxel meshing algorithm than Greedy Meshing, https://stackoverflow.com/questions/72388961/are-there-other-better-voxel-meshing-algorithm-than-greedy-meshing, fetched 2026-10-26 (unofficial)
[S2] Voxel Face Crawling (Mesh simplification, possibly using greedy), https://gamedev.stackexchange.com/questions/58527/voxel-face-crawling-mesh-simplification-possibly-using-greedy, fetched 2026-10-26 (unofficial)
[S3] Meshing in a voxel game, https://stackoverflow.com/questions/49057456/meshing-in-a-voxel-game, fetched 2026-10-26 (unofficial)
[S4] Homebrew Voxel Engine, http://weigert.vsos.ethz.ch/2019/10/27/homebrew-voxel-engine/, fetched 2026-10-26 (unofficial)
[S5] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S6] roboleary/GreedyMesh, https://github.com/roboleary/GreedyMesh, fetched 2026-10-26 (unofficial)
[S7] AThilenius/greedy_meshing, https://github.com/AThilenius/greedy_meshing, fetched 2026-10-26 (unofficial)
[S8] ninthworld/GreedyMeshing, https://github.com/ninthworld/GreedyMeshing, fetched 2026-10-26 (unofficial)
[S9] gilchristb78/Greedy-Meshing-Reloaded, https://github.com/gilchristb78/Greedy-Meshing-Reloaded, fetched 2026-10-26 (unofficial)
[S10] Armen138/greedy_quads_webgl, https://github.com/Armen138/greedy_quads_webgl, fetched 2026-10-26 (unofficial)
[S11] RandomGamingDev/greedy-meshing-js-demo, https://github.com/RandomGamingDev/greedy-meshing-js-demo, fetched 2026-10-26 (unofficial)
[S12] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S13] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
[S14] nwfella/GreedMesh, https://github.com/nwfella/GreedMesh, fetched 2026-10-26 (unofficial)
[S15] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S16] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S17] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
[S19] Greedy Meshing for Vertex Colored Voxels In Unity, https://eddieabbondanz.io/post/voxel/greedy-mesh/, fetched 2026-10-27 (unofficial)
[S20] GitHub - BertilBraun/Greedy-Meshing-in-Python: A simple 2D and 3D ..., https://github.com/BertilBraun/Greedy-Meshing-in-Python, fetched 2026-10-27 (unofficial)
[S21] Greedy Meshing Algorithm for Voxel Terrain in Unity, https://unityqueen.com/2026/07/01/greedy-meshing-algorithm-for-voxel-terrain-in-unity/, fetched 2026-10-27 (unofficial)
[S22] Optimizing Voxels - Greedy Meshing pt. 1 - zenny3d.com, https://zenny3d.com/blog/2025/optimizing-voxels-greedy-meshing-pt.-1.html, fetched 2026-10-27 (unofficial)
[S23] Greedy meshing for complex voxel data, https://hub.jmonkeyengine.org/t/greedy-meshing-for-complex-voxel-data/28017, fetched 2026-10-27 (unofficial)
[S24] Greedy Meshing Algorithm, https://github.com/WrackedFella/moho/issues/26, fetched 2026-10-27 (unofficial)
[S25] GitHub - bonsairobo/block-mesh-rs: Generate voxel block meshes in Rust ..., https://github.com/bonsairobo/block-mesh-rs, fetched 2026-10-27 (unofficial)
[S27] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
[S28] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S29] kotychok/mutiny, https://github.com/kotychok/mutiny, fetched 2026-10-27 (unofficial)
[S30] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S31] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
[S32] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
[S33] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S34] DanielWLiu07/voxel-engine, https://github.com/DanielWLiu07/voxel-engine, fetched 2026-10-26 (unofficial)
[S35] Baking Dynamic Ambient Occlusion in Procedural Voxels, https://salivity.github.io/game-development/article/baking-dynamic-ambient-occlusion-in-procedural-voxels, fetched 2026-10-29 (unofficial)
