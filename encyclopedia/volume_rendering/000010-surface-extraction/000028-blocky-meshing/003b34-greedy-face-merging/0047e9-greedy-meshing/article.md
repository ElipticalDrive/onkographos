# Greedy Meshing

Also called: 3D Greedy Meshing, GreedMesh, Greedy Mesher, Greedy Quad Merging, Greedy Quad Meshing, Greedy Voxel Meshing, Voxel Quad Merging

## Overview
Greedy Meshing is a surface extraction algorithm that generates a minimal triangle mesh from voxel data by merging adjacent coplanar faces of identical material into the largest possible axis-aligned quads [S1, S3, S4, S7, S27, unofficial]. It solves the problem of excessive vertex and draw call counts in naive voxel rendering (where every exposed face is a separate quad) by reducing geometry by 1.88×–2.60× in triangle count and up to 54.5× in memory compared to methods like Marching Cubes, while maintaining a blocky aesthetic [S3, S29, unofficial]. It sets itself apart from its parent [Greedy face merging](8935-greedy-face-merging.md) by explicitly using a greedy expansion strategy to maximize quad size within each 2D slice, and from siblings like [Greedy Meshing for Opaque/Undefined Blocks](4917-greedy-meshing-for-opaque-undefined-blocks.md) by its general applicability to multi-material volumes rather than just binary or opaque-specific cases [S3, S14, unofficial].

## How it works
**Inputs:** A 3D voxel volume defined by dimensions $D = [D_x, D_y, D_z]$ and a voxel access function $V(x, y, z)$ returning the material ID (or occupancy) at coordinates $(x, y, z)$. A `mergeKey` function $K(m, x, y, z)$ that maps a material and position to a merge identifier (default: $K(m, x, y, z) = m$) [S13, S26, unofficial]. A `boundaryPolicy` determining if out-of-bounds neighbors are treated as air or solid [S13, unofficial].

**Outputs:** A list of quads, where each quad is defined by an origin $(x_0, y_0, z_0)$, a normal direction $n \in \{0, 1, 2\}$ (X, Y, Z), and dimensions $(w, h)$ in the tangent and bitangent axes [S27, unofficial].

**Data Structures:**
*   **Mask2D:** A 2D boolean or integer array of size $D_t \times D_b$ (where $t$ is tangent, $b$ is bitangent). Stored as a row-major flat array `data` of size $D_t \times D_b$. Access via `get(t, b)` returns `data[b * D_t + t]` [S7, S15, unofficial].
*   **Quad:** A struct containing `origin` (3D vector), `normal` (axis index), `width` (int), `height` (int), and `material_id` (int) [S27, S29, unofficial].

**Algorithm Steps:**

1.  **Initialize Output:** Create an empty list `quads` [S7, unofficial].
2.  **Sweep Axes:** Iterate over the three principal axes for the normal direction $n \in \{0, 1, 2\}$ [S1, S7, unofficial].
    *   Calculate tangent $t = (n + 1) \pmod 3$ and bitangent $b = (n + 2) \pmod 3$ [S1, S7, unofficial].
    *   Define the normal vector $N$ as a unit vector with 1 at index $n$ and 0 elsewhere [S7, unofficial].
3.  **Sweep Slices:** For each slice index $s$ from $0$ to $D_n - 1$:
    *   Initialize a `Mask2D` of size $D_t \times D_b$ to false/zero [S7, unofficial].
    *   **Visibility Check:** For each cell $(t, b)$ in the mask:
        *   Determine the voxel position in the current slice: $P_{curr} = (x, y, z)$ where the $n$-coordinate is $s$, and $t, b$ coordinates are $t, b$ [S7, unofficial].
        *   Determine the neighbor position in the negative normal direction: $P_{prev} = P_{curr} - N$ [S7, unofficial].
        *   Sample $V_{curr} = V(P_{curr})$ and $V_{prev} = V(P_{prev})$ [S7, unofficial].
        *   Handle boundaries: If $P_{prev}$ is out of bounds, treat $V_{prev}$ as air (0) or solid based on `boundaryPolicy` (default: air) [S13, S27, unofficial].
        *   Compute visibility: If $V_{curr} \neq V_{prev}$, the face is exposed. Set `mask.get(t, b)` to $K(V_{curr}, P_{curr})$ (the merge key) [S7, S29, unofficial]. Otherwise, set to 0/false [S27, unofficial].
    *   **Greedy Expansion:** Scan the mask in lexicographic order (row by row, then column by row, i.e., $b$ from $0$ to $D_b-1$, $t$ from $0$ to $D_t-1$) [S27, S29, unofficial].
        *   Let current cell be $(t_0, b_0)$. If `mask.get(t_0, b_0)` is 0/false, continue [S7, unofficial].
        *   Let $key = \text{mask.get}(t_0, b_0)$ [S29, unofficial].
        *   **Expand Width:** Initialize $w = 1$. While $t_0 + w < D_t$ and `mask.get(t_0 + w, b_0)` equals $key$, increment $w$ [S7, S29, unofficial].
        *   **Expand Height:** Initialize $h = 1$. While $b_0 + h < D_b$:
            *   Check if the entire row segment from $t_0$ to $t_0 + w - 1$ at bitangent $b_0 + h$ has the value $key$.
            *   If any cell in this segment is not $key$, break [S7, unofficial].
            *   Otherwise, increment $h$ [S7, unofficial].
        *   **Emit Quad:** Create a quad with:
            *   Origin: $(x, y, z)$ where $n$-coord is $s$, $t$-coord is $t_0$, $b$-coord is $b_0$ [S7, unofficial].
            *   Normal: $n$ [S27, unofficial].
            *   Dimensions: width $w$, height $h$ [S7, unofficial].
            *   Material: $key$ [S29, unofficial].
            *   Add to `quads` [S7, unofficial].
        *   **Mark Processed:** Set `mask.get(t, b)` to 0/false for all $t \in [t_0, t_0 + w)$ and $b \in [b_0, b_0 + h)$ to prevent double-counting [S27, S29, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh(volume, dims, boundary_policy="air"):
    """
    volume: function (x, y, z) -> int (material ID)
    dims: tuple (Dx, Dy, Dz)
    boundary_policy: "air" or "solid"
    """
    quads = []
    
    # Iterate over the 3 normal axes
    for n in range(3):
        t = (n + 1) % 3
        b = (n + 2) % 3
        
        # Iterate over slices along the normal axis
        for s in range(dims[n]):
            # Initialize mask for this slice
            # Mask stores the merge key if visible, 0 otherwise
            mask = [[0 for _ in range(dims[t])] for _ in range(dims[b])]
            
            # Build the visibility mask
            for y in range(dims[b]):
                for x in range(dims[t]):
                    # Construct current voxel coordinates
                    curr = [0, 0, 0]
                    curr[n] = s
                    curr[t] = x
                    curr[b] = y
                    
                    # Construct previous voxel coordinates (in -normal direction)
                    prev = [c - 1 if i == n else c for i, c in enumerate(curr)]
                    
                    # Sample voxels
                    try:
                        v_curr = volume(curr[0], curr[1], curr[2])
                    except IndexError:
                        v_curr = 0 # Out of bounds
                        
                    try:
                        v_prev = volume(prev[0], prev[1], prev[2])
                    except IndexError:
                        # Boundary handling
                        if boundary_policy == "solid":
                            v_prev = 1 # Treat as solid
                        else:
                            v_prev = 0 # Treat as air (default)
                            
                    # Visibility check: face exists if materials differ
                    if v_curr != v_prev:
                        mask[y][x] = v_curr # Use material ID as merge key
                    
            # Greedy expansion on the mask
            for y in range(dims[b]):
                for x in range(dims[t]):
                    if mask[y][x] == 0:
                        continue
                        
                    key = mask[y][x]
                    
                    # Expand width
                    w = 1
                    while x + w < dims[t] and mask[y][x + w] == key:
                        w += 1
                        
                    # Expand height
                    h = 1
                    while y + h < dims[b]:
                        # Check if the entire row segment matches
                        row_match = True
                        for k in range(w):
                            if mask[y + h][x + k] != key:
                                row_match = False
                                break
                        if not row_match:
                            break
                        h += 1
                        
                    # Emit quad
                    origin = [0, 0, 0]
                    origin[n] = s
                    origin[t] = x
                    origin[b] = y
                    
                    quads.append({
                        "origin": origin,
                        "normal": n,
                        "width": w,
                        "height": h,
                        "material": key
                    })
                    
                    # Mark as processed
                    for dy in range(h):
                        for dx in range(w):
                            mask[y + dy][x + dx] = 0
                            
    return quads
```

**Parameters:**
*   `boundary_policy`: "air" (default) or "solid". Determines if the outer surface of the volume is emitted [S13, unofficial].
*   `mergeKey`: Function mapping material to merge ID. Default is identity [S13, unofficial].
*   `dims`: Tuple of integers $(D_x, D_y, D_z)$ [S7, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Integer coordinates are sufficient for block-aligned meshes. Floating point errors are not a factor in the meshing logic itself, but may affect vertex positioning if scaled [S6, unofficial].
*   **Boundaries:** Out-of-bounds access must be handled explicitly. Treating out-of-bounds as air ensures the volume's exterior is meshed; treating as solid hides it [S13, S27, unofficial].
*   **Transparency:** Standard greedy meshing merges opaque faces. Transparent faces are often kept unit-sized to preserve correct sorting and per-vertex attributes (like AO), preventing visual artifacts from merging faces with different lighting or transparency levels [S33, S39, unofficial].

## Subtopics

### [Voxel Mesh Generation (Run-based Greedy Meshing)](3127-voxel-mesh-generation-run-based-greedy-meshing.md)
A CPU-based variant that combines adjacent faces into runs along X and Y axes, optimizing for generation speed over minimal triangle count. It produces meshes with ~20% more triangles than standard greedy meshing but runs ~390% faster, making it suitable for frequent chunk regeneration in multiplayer voxel games.

### [Chunked Voxel Mesh Generation with Face Culling and Quad Expansion](3133-chunked-voxel-mesh-generation-with-face-culling-and-quad-exp.md)
Handles neighbor chunk references to determine visibility across chunk boundaries and uses a packed integer format for vertex attributes. Supports multiple voxel types and ambient occlusion, unlike binary variants.

### [Binary Greedy Meshing with Packed Quads](3312-binary-greedy-meshing-with-packed-quads.md)
Operates on packed binary planes (1 bit per voxel), using CPU-native bitwise operations (trailing zeros, shifts) for greedy expansion. Significantly reduces clock cycles compared to branching logic in standard greedy meshing.

### [Greedy Meshing for Voxel Faces](3367-greedy-meshing-for-voxel-faces.md)
Extends standard greedy meshing by merging faces based on multiple attributes (material, transparency) using a custom equality check, allowing for complex culling and attribute-based optimization.

### [Greedy Meshing with Vertex Pooling](6808-greedy-meshing-with-vertex-pooling.md)
Combines greedy meshing with a memory management system that batches quads from multiple chunks into shared vertex pools, reducing draw calls by an order of magnitude.

### [Greedy Meshing / Face Culling](8783-greedy-meshing-face-culling.md)
Generates static meshes by emitting geometry only for faces adjacent to empty space. Unlike standard greedy meshing, it does not merge adjacent coplanar faces into larger quads, resulting in higher primitive counts but simpler logic.

## Operating characteristics
*   **Time Complexity:** $O(V)$, where $V$ is the number of voxels in the volume. Each voxel is visited a constant number of times (once per axis slice, and once during mask expansion) [S1, S7, unofficial].
*   **Memory Complexity:** $O(D_t \times D_b)$ for the mask per slice, plus $O(Q)$ for the output quads, where $Q$ is the number of generated quads [S7, S27, unofficial].
*   **Performance:** Reduces triangle count by 5–15× compared to naive per-face meshing [S29, unofficial]. In a 5x1x5 chunk scene, implementations report ~140 FPS with V-Sync off [S4, unofficial].
*   **Strengths:** Simple to implement, deterministic, fast generation, significant reduction in draw calls and vertex count [S1, S4, S29, unofficial].
*   **Limitations:** Not optimal for dynamic scenes with frequent individual block changes (requires full or partial chunk regeneration). Does not produce smooth surfaces [S1, S3, unofficial]. Transparent faces are difficult to merge efficiently [S33, unofficial].

## Background
Greedy Meshing was popularized by Mikola Lysenko in his 2012 article "Meshing in a Minecraft Game" [S5, unofficial][S28]. It is an optimization over "Culled Meshing" (which removes internal faces but leaves adjacent faces separate) and "Stupid Meshing" (one cube per voxel) [S1, unofficial]. The algorithm is widely used in voxel engines like Minecraft clones, Noa, and various Unity/Roblox implementations [S5, S23, S24, unofficial].

## Sources
[S1] Are there other/better voxel meshing algorithm than Greedy Meshing, https://stackoverflow.com/questions/72388961/are-there-other-better-voxel-meshing-algorithm-than-greedy-meshing, fetched 2026-10-26 (unofficial)
[S2] Voxel Face Crawling (Mesh simplification, possibly using greedy), https://gamedev.stackexchange.com/questions/58527/voxel-face-crawling-mesh-simplification-possibly-using-greedy, fetched 2026-10-26 (unofficial)
[S3] Meshing in a voxel game, https://stackoverflow.com/questions/49057456/meshing-in-a-voxel-game, fetched 2026-10-26 (unofficial)
[S4] Homebrew Voxel Engine, http://weigert.vsos.ethz.ch/2019/10/27/homebrew-voxel-engine/, fetched 2026-10-26 (unofficial)
[S5] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S7] AThilenius/greedy_meshing, https://github.com/AThilenius/greedy_meshing, fetched 2026-10-26 (unofficial)
[S13] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S27] GitHub - bonsairobo/block-mesh-rs: Generate voxel block meshes in Rust ..., https://github.com/bonsairobo/block-mesh-rs, fetched 2026-10-27 (unofficial)
[S29] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
