# Greedy Meshing

Also called: 3D Greedy Meshing, Face Merging, Greedy Surface Extraction, GreedyMesh, GreedyMesher, Minecraft meshing, Trile Polygonization, rectangle triangulation

## Overview
Greedy Meshing is a surface extraction algorithm that converts a dense 3D voxel grid into an optimized triangle mesh by merging adjacent, coplanar faces of identical material into maximal axis-aligned rectangles (quads) [S1, S3, S4, S6, S9, S11, S14, unofficial]. It solves the problem of excessive vertex counts and draw calls inherent in naive per-voxel rendering, where every visible face of every voxel generates geometry [S3, S19, unofficial][S25]. Unlike simple face culling, which only removes hidden faces but leaves individual unit quads, Greedy Meshing actively merges contiguous blocks of identical material into larger primitives, reducing vertex and triangle counts by 10–20x in typical terrain scenarios [S19, S36, S45, unofficial]. It differs from smooth surface extraction methods like Marching Cubes by preserving the sharp, discrete, axis-aligned boundaries characteristic of voxel-based worlds [S6, S34, unofficial].

## How it works
The algorithm processes the voxel volume by sweeping along each of the three principal axes ($n \in \{0, 1, 2\}$ corresponding to X, Y, Z) [G]. For each normal $n$, the algorithm defines the tangent $t = (n + 1) \pmod 3$ and bitangent $b = (n + 2) \pmod 3$ [G].

### Inputs and Outputs
*   **Input:** A voxel volume accessible via a function `volume(x, y, z)` returning a material ID (or integer type), and dimensions `dims = (Dx, Dy, Dz)` [S1, S10, S11, unofficial].
*   **Input:** A `boundary_policy` ("air" or "solid") determining how out-of-bounds neighbors are treated [S1, S17, unofficial].
*   **Input:** An optional `merge_key` function `(material_id, x, y, z) -> int` that allows merging based on attributes beyond simple material ID (e.g., lighting, UV offset) [S1, S17, unofficial]. Default is identity.
*   **Output:** A list of quads, each defined by an origin $(x, y, z)$, a normal $n$, and dimensions $(w, h)$ [S1, S10, S17, unofficial].

### Data Structures
*   **Mask:** A 2D array of size $D_t \times D_b$ (where $D_t = \text{dims}[t]$ and $D_b = \text{dims}[b]$). It stores the `merge_key` of a visible, unprocessed face, or $0$ if the face is hidden or already processed [S1, S10, S17, unofficial].
*   **Quad Record:** A structure containing `origin` (array of 3 ints), `normal` (int 0-2), `width` (int), and `height` (int) [S1, S10, S17, unofficial].

### Steps
1.  **Initialize:** Create an empty list `quads`.
2.  **Sweep Axes:** Iterate $n$ from 0 to 2.
    *   Set $t = (n + 1) \pmod 3$ and $b = (n + 2) \pmod 3$.
    *   Initialize `mask` as a 2D array of zeros with dimensions $D_t \times D_b$.
3.  **Sweep Slices:** Iterate slice index $s$ from 0 to $D_n - 1$.
    *   **Build Visibility Mask:** For each cell $(x, y)$ in the 2D plane (where $x$ corresponds to axis $t$ and $y$ to axis $b$):
        *   Construct coordinates `coords` such that `coords[n] = s`, `coords[t] = x`, `coords[b] = y`.
        *   Determine if the face is exposed:
            *   Check neighbors in the normal direction: $s-1$ and $s+1$.
            *   A face is exposed if the voxel at `coords` has a different material ID than its neighbor, or if the neighbor is out of bounds and `boundary_policy` is "air" [S1, S10, S17, unofficial].
            *   If exposed, set `mask[y * D_t + x] = merge_key(volume(*coords), *coords)`.
            *   Otherwise, set `mask[y * D_t + x] = 0`.
    *   **Greedy Expansion:** Scan the mask in lexicographic order (row $y$ from 0 to $D_b-1$, column $x$ from 0 to $D_t-1$).
        *   If `mask[y * D_t + x]` is 0, continue.
        *   Let `key = mask[y * D_t + x]`. This is the start of a new quad.
        *   **Expand Width ($w$):** Increment $w$ starting at 1 while $x + w < D_t$ and `mask[y * D_t + (x + w)] == key`.
        *   **Expand Height ($h$):** Increment $h$ starting at 1 while $y + h < D_b$. For each potential $h$, check if all cells in the row $y+h$ from column $x$ to $x+w-1$ have `mask[(y+h) * D_t + k] == key` for $k \in [x, x+w)$. If any cell mismatches, stop expanding height.
        *   **Emit Quad:** Create a quad with:
            *   `origin[n] = s`, `origin[t] = x`, `origin[b] = y`.
            *   `normal = n`.
            *   `width = w`, `height = h`.
            *   Add to `quads`.
        *   **Mark Processed:** Set `mask[(y + dy) * D_t + (x + dx)] = 0` for all $0 \le dy < h$ and $0 \le dx < w$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_meshing(volume, dims, boundary_policy="air", merge_key=None):
    """
    volume: function (x, y, z) -> material_id
    dims: tuple (Dx, Dy, Dz)
    boundary_policy: "air" (default) or "solid"
    merge_key: function (material_id, x, y, z) -> int. Default: identity.
    """
    if merge_key is None:
        merge_key = lambda m, x, y, z: m

    quads = []
    # Iterate over the 3 normal axes: 0=X, 1=Y, 2=Z
    for n in range(3):
        t = (n + 1) % 3  # Tangent axis
        b = (n + 2) % 3  # Bitangent axis
        dim_t, dim_b, dim_n = dims[t], dims[b], dims[n]
        
        # Mask stores the merge key if visible and unprocessed, else 0
        mask = [0] * (dim_t * dim_b)
        
        # Iterate through each slice along the normal axis
        for s in range(dim_n):
            # 1. Build Visibility Mask for slice s
            for y in range(dim_b):
                for x in range(dim_t):
                    coords = [0, 0, 0]
                    coords[n] = s
                    coords[t] = x
                    coords[b] = y
                    voxel_val = volume(*coords)
                    
                    exposed = False
                    # Check both neighbors along the normal axis
                    for dir in [-1, 1]:
                        nb_s = s + dir
                        if 0 <= nb_s < dim_n:
                            nb_coords = coords.copy()
                            nb_coords[n] = nb_s
                            nb_val = volume(*nb_coords)
                            if voxel_val != nb_val:
                                exposed = True
                                break
                        else:
                            # Out of bounds neighbor
                            if boundary_policy == "air":
                                exposed = True  # Air neighbor implies exposed face
                                break
                            else:  # solid
                                exposed = False  # Solid neighbor implies internal
                                break
                    
                    if exposed:
                        mask[y * dim_t + x] = merge_key(voxel_val, *coords)
                    else:
                        mask[y * dim_t + x] = 0

            # 2. Greedy Expansion on the 2D Mask
            for y in range(dim_b):
                for x in range(dim_t):
                    idx = y * dim_t + x
                    key = mask[idx]
                    
                    if key == 0:
                        continue
                    
                    # Start of a new quad
                    w = 1
                    # Expand width
                    while x + w < dim_t and mask[y * dim_t + (x + w)] == key:
                        w += 1
                    
                    h = 1
                    # Expand height
                    while y + h < dim_b:
                        row_match = True
                        for k in range(w):
                            if mask[(y + h) * dim_t + (x + k)] != key:
                                row_match = False
                                break
                        if not row_match:
                            break
                        h += 1
                    
                    # Emit Quad
                    origin = [0, 0, 0]
                    origin[n] = s
                    origin[t] = x
                    origin[b] = y
                    quads.append({
                        'origin': origin,
                        'normal': n,
                        'width': w,
                        'height': h
                    })
                    
                    # Mark cells as processed (set to 0)
                    for dy in range(h):
                        for dx in range(w):
                            mask[(y + dy) * dim_t + (x + dx)] = 0
                            
    return quads
```

### Parameters and Edge Cases
*   **Boundary Policy:** If "air", voxels at the chunk boundary are considered exposed to air, generating faces [S1, S17, unofficial]. If "solid", they are internal and hidden [S17, unofficial].
*   **Merge Key:** Using a custom key allows merging faces with different visual properties (e.g., different AO values) if they are deemed identical for rendering purposes [S1, S17, S45, unofficial].
*   **Precision:** The algorithm relies on exact equality checks for material IDs and merge keys. Floating-point coordinates should be avoided for voxel indexing; integer grid coordinates are standard [S37, unofficial].
*   **Degenerate Input:** Empty volumes or volumes with no exposed faces produce an empty quad list [G].

## Subtopics

### [Chunked Voxel Mesh Generation with Face Culling and Quad Expansion](3133-chunked-voxel-mesh-generation-with-face-culling-and-quad-exp.md)
Chunked Voxel Mesh Generation with Face Culling and Quad Expansion is a CPU-based algorithm that converts discrete 3D voxel data into optimized triangle meshes by merging adjacent coplanar faces of identical voxel types into larger quads. It solves the problem of excessive vertex and draw call counts in voxel worlds by reducing primitive density while preserving sharp, axis-aligned surfaces suitable for architectural geometry. It differs from standard face culling (which only removes hidden faces) by actively merging contiguous blocks into maximal axis-aligned rectangles, and it differs from Marching Cubes by maintaining discrete voxel boundaries rather than smoothing surfaces. This method builds on [Greedy Meshing](3519-greedy-meshing.md) by processing data in fixed-size chunks and handling inter-chunk visibility.

### [Greedy Meshing for Voxel Faces](3367-greedy-meshing-for-voxel-faces.md)
Greedy Meshing for Voxel Faces is a mesh simplification technique that reduces the number of triangles and draw calls in voxel rendering by merging adjacent coplanar voxel faces into larger rectangular quads. Unlike standard greedy meshing, which merges voxels based solely on existence or a single block type, this method compares voxel faces based on multiple attributes (such as material type, transparency, and lighting) to determine mergeability. It builds on [Greedy Meshing](18808-greedy-meshing.md) by extending the 2D slice processing to handle face-level attributes, ensuring that only faces with identical attributes are merged.

### [Integrating AO with Greedy Meshing](5550-integrating-ao-with-greedy-meshing.md)
Integrating Ambient Occlusion (AO) with Greedy Meshing resolves the conflict between merging coplanar voxel faces into large quads and the natural variation of occlusion values across individual block boundaries. Unlike standard greedy meshing which prioritizes minimal polygon count, this method enforces strict visual continuity of lighting by ensuring that the interpolated shading on a merged quad accurately represents the occlusion of every unit face it replaces. It achieves this through specific merge rules, compact data packing, or shader-based reconstruction strategies.

### [Greedy Meshing (Back Face)](16006-greedy-meshing-back-face.md)
Greedy Meshing (Back Face) is a specialized surface extraction method that generates quads for the back-facing surfaces (negative Z direction) of a voxel chunk. It extends the 2D greedy meshing algorithm to three dimensions by extracting 2D slices from the 3D voxel grid along the Z-axis and processing them to merge adjacent blocks of the same type into larger rectangles. It is distinct from general greedy meshing [18808-greedy-meshing.md] because it operates exclusively on the back-facing boundary, requiring specific handling of neighbor chunks to determine visibility at the chunk's rear edge.

### [VoxelMesherCubes](19075-voxelmeshercubes.md)
VoxelMesherCubes generates a mesh representing cubic voxels directly from color values stored in a `VoxelBuffer.CHANNEL_COLOR` channel. Unlike its parent [Greedy Meshing](18808-greedy-meshing.md), which optimizes for triangle count via quad merging, VoxelMesherCubes focuses on faithful geometric representation of individual cubes, though it optionally supports greedy merging. It differs from [Voxel Meshing via Column Flushing](21370-voxel-meshing-via-column-flushing.md) by generating individual cube faces rather than merged column segments, making it suitable for standard vertex color rendering without custom shader logic for color mapping.

### SBUG Mesh Entry Point
The SBUG Mesh Entry Point defines the interface for streaming binary greedy meshing. It initializes output buffers for 6 face directions, resizes them to a maximum capacity of $CS^3$ quads, and calls the core meshing function `mesh_face_columns_all_x_slices`. It then calculates the final vertex counts per face and returns the total vertex count [S52, unofficial].

### 4D Greedy Meshing
4D Greedy Meshing applies the greedy merging principle to 4D voxel worlds. It sweeps in 4D lattice space rather than 3D slice space, leveraging the property that the preimage of a convex set under a linear map is convex. This allows merging coplanar faces across different angles in slice space. Re-merging in a third dimension recovers only about 1% of quads at any cut, contrary to earlier design notes claiming no further merging was possible [S53, unofficial]. Reduction ratios compared to naive per-face meshing range from 1.41x (flat cut) to 5.3x overall [S53, unofficial].

### VBO Indexing Optimization
VBO Indexing Optimization is a post-processing step applied after greedy meshing. It reduces the vertex count from 6 vertices per quad (if using triangles) to 4 vertices per quad by sharing vertices among adjacent quads in the same plane, further reducing the quad count by 1/3 compared to the greedy meshing output alone [S4, unofficial].

### Padded Chunk Volume Meshing
Padded Chunk Volume Meshing meshes a 32³ chunk using a 34³ padded volume that includes its 26 immediate neighbors. This ensures seamless borders and per-vertex ambient occlusion in all three axes, preventing visual artifacts at chunk boundaries [S54, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V)$, where $V$ is the number of voxels in the chunk. The algorithm visits each voxel a constant number of times (once per axis slice) [S1, S10, S17, unofficial].
*   **Memory Complexity:** $O(D_t \times D_b)$ for the mask, which is at most $O(D^2)$ for a chunk of size $D$. This is significantly smaller than storing a full mesh [S1, S17, unofficial].
*   **Performance:** Reduces triangle count by 10–20x compared to naive per-face rendering [S19, S36, unofficial]. In typical Minecraft-like terrain, it reduces draw calls by 60% or more [S4, unofficial].
*   **Strengths:** Simple to implement, fast, deterministic, preserves sharp voxel aesthetics [S6, S31, unofficial].
*   **Limitations:** Does not handle transparency well (transparent faces are often kept unit-sized to preserve sorting order) [S45, unofficial]. Merging across different material attributes (e.g., AO) requires careful key design [S45, unofficial]. Not suitable for smooth surfaces [S6, unofficial].

## Background
Greedy Meshing was popularized by Mikola Lysenko in his 2012 article "Meshing in a Minecraft Game" [S5, unofficial][S35][S10, unofficial]. It emerged as a solution to the performance bottlenecks of early voxel engines, which rendered every visible face of every voxel. The algorithm is now a standard technique in voxel game development, used in engines like Minecraft, Noa, and various custom engines [S5, S7, S10, unofficial].

## Sources
[S1] Are there other/better voxel meshing algorithm than Greedy Meshing, https://stackoverflow.com/questions/72388961/are-there-other-better-voxel-meshing-algorithm-than-greedy-meshing, fetched 2026-10-26 (unofficial)
[S2] Voxel Face Crawling (Mesh simplification, possibly using greedy), https://gamedev.stackexchange.com/questions/58527/voxel-face-crawling-mesh-simplification-possibly-using-greedy, fetched 2026-10-26 (unofficial)
[S3] Meshing in a voxel game, https://stackoverflow.com/questions/49057456/meshing-in-a-voxel-game, fetched 2026-10-26 (unofficial)
[S4] Homebrew Voxel Engine, http://weigert.vsos.ethz.ch/2019/10/27/homebrew-voxel-engine/, fetched 2026-10-26 (unofficial)
[S5] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S6] How can I draw multiple(10000) cubes in the bevy game engine for my 3D voxel game?, https://stackoverflow.com/questions/66609344/how-can-i-draw-multiple10000-cubes-in-the-bevy-game-engine-for-my-3d-voxel-gam, fetched 2026-10-26 (unofficial)
[S7] Voxel graphics: MagicaVoxel import, animation, dynamic world and physics (ADR-0025), https://github.com/AdaEngine/Ada/pull/136, fetched 2026-10-26 (unofficial)
[S8] roboleary/GreedyMesh, https://github.com/roboleary/GreedyMesh, fetched 2026-10-26 (unofficial)
[S9] knaxel/Voxel-Engine-Example, https://github.com/knaxel/Voxel-Engine-Example, fetched 2026-10-26 (unofficial)
[S10] AThilenius/greedy_meshing, https://github.com/AThilenius/greedy_meshing, fetched 2026-10-26 (unofficial)
[S11] wwwredfish/CEWBS, https://github.com/wwwredfish/CEWBS, fetched 2026-10-26 (unofficial)
[S12] ninthworld/GreedyMeshing, https://github.com/ninthworld/GreedyMeshing, fetched 2026-10-26 (unofficial)
[S13] gilchristb78/Greedy-Meshing-Reloaded, https://github.com/gilchristb78/Greedy-Meshing-Reloaded, fetched 2026-10-26 (unofficial)
[S14] Armen138/greedy_quads_webgl, https://github.com/Armen138/greedy_quads_webgl, fetched 2026-10-26 (unofficial)
[S15] RandomGamingDev/greedy-meshing-js-demo, https://github.com/RandomGamingDev/greedy-meshing-js-demo, fetched 2026-10-26 (unofficial)
[S16] lazergenixdev/First-Voxel-Engine, https://github.com/lazergenixdev/First-Voxel-Engine, fetched 2026-10-26 (unofficial)
[S17] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S18] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
[S19] nwfella/GreedMesh, https://github.com/nwfella/GreedMesh, fetched 2026-10-26 (unofficial)
[S20] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S21] BLaZeKiLL/VloxyEngine, https://github.com/BLaZeKiLL/VloxyEngine, fetched 2026-10-26 (unofficial)
[S22] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S23] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
[S24] Optimising Voxel World Rendering, https://gamedev.stackexchange.com/questions/117398/optimising-voxel-world-rendering, fetched 2026-10-27 (unofficial)
[S26] Greedy Meshing for Vertex Colored Voxels In Unity, https://eddieabbondanz.io/post/voxel/greedy-mesh/, fetched 2026-10-27 (unofficial)
[S27] GitHub - BertilBraun/Greedy-Meshing-in-Python: A simple 2D and 3D ..., https://github.com/BertilBraun/Greedy-Meshing-in-Python, fetched 2026-10-27 (unofficial)
[S28] Greedy Meshing Algorithm for Voxel Terrain in Unity, https://unityqueen.com/2026/07/01/greedy-meshing-algorithm-for-voxel-terrain-in-unity/, fetched 2026-10-27 (unofficial)
[S29] Greedy Meshing Voxels - Community Resources - Roblox, https://devforum.roblox.com/t/greedy-meshing-voxels/1139881, fetched 2026-10-27 (unofficial)
[S30] Consume everything - how greedy meshing works - Roblox, https://devforum.roblox.com/t/consume-everything-how-greedy-meshing-works/452717, fetched 2026-10-27 (unofficial)
[S31] Optimizing Voxels - Greedy Meshing pt. 1 - zenny3d.com, https://zenny3d.com/blog/2025/optimizing-voxels-greedy-meshing-pt.-1.html, fetched 2026-10-27 (unofficial)
[S32] Greedy meshing for complex voxel data, https://hub.jmonkeyengine.org/t/greedy-meshing-for-complex-voxel-data/28017, fetched 2026-10-27 (unofficial)
[S33] Greedy Meshing Algorithm, https://github.com/WrackedFella/moho/issues/26, fetched 2026-10-27 (unofficial)
[S34] GitHub - bonsairobo/block-mesh-rs: Generate voxel block meshes in Rust ..., https://github.com/bonsairobo/block-mesh-rs, fetched 2026-10-27 (unofficial)
[S36] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
[S37] GreedyMesher: voxel mesh compression for Metal instanced rendering, https://github.com/plagness/VoxelKit/issues/13, fetched 2026-10-27 (unofficial)
[S38] Gamengineer314/GLVoxelTerrain, https://github.com/Gamengineer314/GLVoxelTerrain, fetched 2026-10-27 (unofficial)
[S39] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S40] zer0nim/ft_vox, https://github.com/zer0nim/ft_vox, fetched 2026-10-27 (unofficial)
[S41] kotychok/mutiny, https://github.com/kotychok/mutiny, fetched 2026-10-27 (unofficial)
[S42] Greedy Meshing - GitHub, https://github.com/programmer1o1/GreedyMeshingMod, fetched 2026-10-27 (unofficial)
[S45] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S46] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
[S47] Behind Fez : Trixels (part two) – The Instruction Limit, https://theinstructionlimit.com/behind-fez-trixels-part-two, fetched 2026-10-28 (unofficial)
[S48] all voxel rendering techniques : r/VoxelGameDev, https://www.reddit.com/r/VoxelGameDev/comments/11az6wm/all_voxel_rendering_techniques/, fetched 2026-10-28 (unofficial)
[S49] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
[S51] Greedy meshing, visually, https://fluff.blog/2023/04/24/greedy-meshing-visually.html, fetched 2026-10-30 (unofficial)
[S52] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S53] DanielWLiu07/voxel-engine, https://github.com/DanielWLiu07/voxel-engine, fetched 2026-10-26 (unofficial)
[S54] 12.8B destructible voxels rendered in the browser with Rust WASM custom engine, https://github.com/cktang88/2.5d-experiment, fetched 2026-10-26 (unofficial)
