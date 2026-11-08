# Quad-to-Geometry Assembly

Also called: Quad BufferGeometry Conversion, quadsToGeometry

## Overview
Quad-to-Geometry Assembly converts a list of axis-aligned quads into flat, GPU-ready vertex and index buffers for WebGL/OpenGL rendering [S1, unofficial]. It differs from sibling methods like [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md) by pre-computing all vertex data on the CPU rather than generating geometry in the vertex shader [G].

## How it works
**Inputs:**
- `quads`: An array of quad objects. Each quad contains `axis` (0=X, 1=Y, 2=Z), `dir` (winding orientation), `slice` (coordinate perpendicular to face), `u` and `v` (start coordinates parallel to face), `w` and `h` (width and height in voxel units), and `material` (integer index) [S1, unofficial].
- `scale`: A global multiplier for position coordinates (default 1.0) [S1, unofficial].

**Outputs:**
- `positions`: `Float32Array` of size `quads.length * 12` [S1, unofficial].
- `normals`: `Float32Array` of size `quads.length * 12` [S1, unofficial].
- `colors`: `Float32Array` of size `quads.length * 12` [S1, unofficial].
- `indices`: `Uint16Array` or `Uint32Array` of size `quads.length * 6` [S1, unofficial].

**Data Structures:**
The output arrays are flat and interleaved by attribute type, not by vertex [S1, unofficial].
- `positions`, `normals`, `colors`: Each stores 3 floats per vertex, totaling 12 floats per quad [S1, unofficial].
- `indices`: Stores 6 indices per quad (two triangles) [S1, unofficial].

**Material Colors:**
A lookup table `MATERIAL_COLORS` maps material indices to RGB values [S1, unofficial]:
- Index 0: `[0.0, 0.0, 0.0]` (Air/Unused)
- Index 1: `[0.24, 0.72, 0.48]` (1-surface)
- Index 2: `[0.46, 0.33, 0.23]` (2-dirt)
- Index 3: `[0.44, 0.47, 0.55]` (3-stone)
- Index 4: `[0.13, 0.83, 0.93]` (4-crystal) [S1, unofficial].

**Algorithm Steps:**
1. Calculate `vertexCount = quads.length * 4` [S1, unofficial].
2. Determine index type: if `vertexCount > 65535`, allocate `indices` as `Uint32Array`; otherwise, allocate as `Uint16Array` [S1, unofficial].
3. Initialize `positions`, `normals`, and `colors` as `Float32Array` of size `quads.length * 12` [S1, unofficial].
4. Initialize `indices` as the determined type of size `quads.length * 6` [S1, unofficial].
5. Iterate through each quad `q` with index `i`:
   a. Compute base position `p` using `q.axis`, `q.slice`, `q.u`, `q.v` [S1, unofficial].
   b. Compute direction vectors `du` and `dv` using `q.w`, `q.h`, `q.axis`, and `q.dir` [S1, unofficial].
   c. Define four vertices: `v0 = p`, `v1 = p + du`, `v2 = p + du + dv`, `v3 = p + dv` [S1, unofficial].
   d. Scale positions by `scale` and write to `positions` at offsets `po` to `po + 11` [S1, unofficial].
   e. Set normals to axis-aligned unit vectors based on `q.axis` and `q.dir`, write to `normals` [S1, unofficial].
   f. Look up `q.material` in `MATERIAL_COLORS` (default to index 3 if out of bounds), write to `colors` [S1, unofficial].
   g. Generate 6 indices using `quadTriangleIndices(q.dir)`, offset by `base`, write to `indices` at `io` [S1, unofficial].
   h. Increment `base` by 4, `po` by 12, `io` by 6 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quad_to_geometry(quads, scale=1.0):
    num_quads = len(quads)
    num_vertices = num_quads * 4
    num_indices = num_quads * 6

    positions = [0.0] * (num_quads * 12)
    normals = [0.0] * (num_quads * 12)
    colors = [0.0] * (num_quads * 12)
    
    if num_vertices > 65535:
        indices = [0] * num_indices  # Uint32
    else:
        indices = [0] * num_indices  # Uint16

    material_colors = [
        [0.0, 0.0, 0.0],      # 0: Air
        [0.24, 0.72, 0.48],   # 1: Surface
        [0.46, 0.33, 0.23],   # 2: Dirt
        [0.44, 0.47, 0.55],   # 3: Stone
        [0.13, 0.83, 0.93]    # 4: Crystal
    ]

    base = 0
    po = 0
    io = 0

    for q in quads:
        # 1. Compute base position p
        if q.axis == 0:
            p = [q.slice, q.u, q.v]
        elif q.axis == 1:
            p = [q.u, q.slice, q.v]
        else:
            p = [q.u, q.v, q.slice]

        # 2. Compute direction vectors du and dv
        du = [0.0, 0.0, 0.0]
        dv = [0.0, 0.0, 0.0]
        if q.axis == 0:
            du[1] = q.w if q.dir == 0 else -q.w
            du[2] = q.h if q.dir == 0 else -q.h
            dv[1] = q.h if q.dir == 0 else -q.h
            dv[2] = q.w if q.dir == 0 else -q.w
        elif q.axis == 1:
            du[0] = q.w if q.dir == 0 else -q.w
            du[2] = q.h if q.dir == 0 else -q.h
            dv[0] = q.h if q.dir == 0 else -q.h
            dv[2] = q.w if q.dir == 0 else -q.w
        else:
            du[0] = q.w if q.dir == 0 else -q.w
            du[1] = q.h if q.dir == 0 else -q.h
            dv[0] = q.h if q.dir == 0 else -q.h
            dv[1] = q.w if q.dir == 0 else -q.w

        # 3. Compute 4 vertices
        v0 = p
        v1 = [p[0]+du[0], p[1]+du[1], p[2]+du[2]]
        v2 = [v1[0]+dv[0], v1[1]+dv[1], v1[2]+dv[2]]
        v3 = [p[0]+dv[0], p[1]+dv[1], p[2]+dv[2]]

        # 4. Write positions (scaled)
        for j, v in enumerate([v0, v1, v2, v3]):
            for k in range(3):
                positions[po + j*3 + k] = v[k] * scale

        # 5. Write normals
        normal = [0.0, 0.0, 0.0]
        normal[q.axis] = 1.0 if q.dir == 0 else -1.0
        for j in range(4):
            for k in range(3):
                normals[po + j*3 + k] = normal[k]

        # 6. Write colors
        mat_idx = q.material
        if mat_idx < 0 or mat_idx >= len(material_colors):
            mat_idx = 3
        color = material_colors[mat_idx]
        for j in range(4):
            for k in range(3):
                colors[po + j*3 + k] = color[k]

        # 7. Write indices
        tri_indices = [0, 1, 2, 0, 2, 3] if q.dir == 0 else [0, 2, 1, 0, 3, 2]
        for j in range(6):
            indices[io + j] = base + tri_indices[j]

        base += 4
        po += 12
        io += 6

    return positions, normals, colors, indices
```

**Edge Cases and Pitfalls:**
- **Index Overflow:** If `vertexCount > 65535`, using `Uint16Array` causes silent wrapping and giant triangle spikes. The method checks this condition and switches to `Uint32Array` [S1, unofficial].
- **Material Index Bounds:** If `q.material` is out of bounds for `MATERIAL_COLORS`, it defaults to index 3 (stone) to prevent errors [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of quads, as the algorithm iterates through the quad list once [S1, unofficial].
- **Memory Complexity:** $O(N)$, allocating four arrays proportional to the number of quads [S1, unofficial].
- **Strengths:** Simple, deterministic, and produces flat buffers suitable for direct GPU upload. Efficient for static geometry [S1, unofficial].
- **Limitations:** High memory usage for large scenes. Requires careful index type handling to avoid overflow. Not optimized for dynamic updates without full array rebuilds [S1, unofficial].

## Background
This method is associated with TypeScript implementations of voxel greedy meshing, specifically found in the `m2-md/voxel-greedy-meshing-typescript` repository [S1, unofficial].

## Sources
[S1] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
