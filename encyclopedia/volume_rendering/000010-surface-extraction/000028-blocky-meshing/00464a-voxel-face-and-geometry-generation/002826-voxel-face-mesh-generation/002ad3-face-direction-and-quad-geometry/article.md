# Face Direction and Quad Geometry

Also called: FaceDir, Index Generation, Quad Position Generation, UV Generation, Standard Vertex Buffer Rendering (Pipeline B/C)

## Overview
Face Direction and Quad Geometry defines the six axis-aligned face orientations on a voxel cube and provides the deterministic logic for generating quad geometry (positions, indices, UVs, and normals) for meshing [S1, unofficial]. It solves the problem of converting abstract voxel face exposure data into concrete vertex buffer layouts suitable for standard rasterization pipelines [S2, unofficial].

## How it works
**Inputs:**
- `FaceDir`: An integer enum representing one of the six face orientations.
- `voxel`: A 3D integer coordinate `[u32; 3]` representing the minimum-corner voxel of the quad in padded chunk coordinates [S1, unofficial].
- `w`, `h`: Unsigned integers representing the width and height of the quad in voxel units [S1, unofficial].
- `base_idx`: The starting index of the four vertices in the vertex buffer [S1, unofficial].
- `flip_u`, `flip_v`: Boolean flags indicating whether to flip the U or V texture coordinates [S1, unofficial].

**Outputs:**
- A set of four vertex positions in local block space.
- A set of four UV coordinates.
- An index buffer containing six indices forming two triangles.
- A normal vector for the face.

**Data Structures:**

`FaceDir` enum (mapped to `u8`):
- `XNeg = 0`
- `XPos = 1`
- `YNeg = 2`
- `YPos = 3`
- `ZNeg = 4`
- `ZPos = 5`

The discriminants match the index in a `[Vec<Quad>; 6]` buffer [S1, unofficial].

`Quad` struct:
- `voxel: [u32; 3]`: Minimum-corner voxel in padded chunk coordinates [S1, unofficial].
- `width: u32` [S1, unofficial].
- `height: u32` [S1, unofficial].

`Vertex` struct:
- `x, y, z: float` [S2, unofficial].
- `u, v: float` [S2, unofficial].

**Step-by-Step Method:**

1.  **Normal Calculation**: Determine the integer normal vector $\mathbf{n}$ based on `FaceDir` [S1, unofficial]:
    -   `XNeg`: $\mathbf{n} = [-1, 0, 0]$
    -   `XPos`: $\mathbf{n} = [1, 0, 0]$
    -   `YNeg`: $\mathbf{n} = [0, -1, 0]$
    -   `YPos`: $\mathbf{n} = [0, 1, 0]$
    -   `ZNeg`: $\mathbf{n} = [0, 0, -1]$
    -   `ZPos`: $\mathbf{n} = [0, 0, 1]$

2.  **Axis Identification**: Determine the axes for the local UV space [S1, unofficial]:
    -   `normal_axis()`: Returns 0 for X faces, 1 for Y faces, 2 for Z faces.
    -   `u_axis()`: Returns 2 (Z) for X faces, 2 (Z) for Y faces, 0 (X) for Z faces.
    -   `v_axis()`: Returns 1 (Y) for X faces, 0 (X) for Y faces, 1 (Y) for Z faces.

3.  **Quad Position Generation**: Calculate the local block position for the four vertices of the quad [S1, S2, unofficial].
    -   Determine the `origin` in block space. If the face is positive (`is_positive()` is true), add $1.0$ to the coordinate corresponding to `normal_axis()`. Otherwise, use the `voxel` coordinate for that axis [S1, unofficial].
    -   The four vertices in local block space (0-1) are defined as:
        -   $V_0$: `min_u_min_v` (origin)
        -   $V_1$: `max_u_min_v` (origin + $w$ on `u_axis`)
        -   $V_2$: `min_u_max_v` (origin + $h$ on `v_axis`)
        -   $V_3$: `max_u_max_v` (origin + $w$ on `u_axis` + $h$ on `v_axis`)

4.  **Index Generation**: Generate two triangles for the quad [S1, S2, unofficial].
    -   Let `base` be the starting index of the four vertices in the vertex buffer.
    -   If the face is `XNeg`, `YPos`, or `ZPos`, use Counter-Clockwise (CCW) winding:
        -   Indices: `[base, base+1, base+2, base+1, base+3, base+2]`
    -   Otherwise (for `XPos`, `YNeg`, `ZNeg`), use Clockwise (CW) winding:
        -   Indices: `[base, base+2, base+1, base+1, base+2, base+3]`
    -   Alternatively, push 6 indices forming two triangles: `startIndex+0,1,2` and `startIndex+2,3,0` [S2, unofficial].

5.  **UV Generation**: Generate UV coordinates for the four vertices [S1, S2, unofficial].
    -   Define local UVs for corners: $(0,0), (0,1), (1,1), (1,0)$ corresponding to vertex order $V_0, V_1, V_2, V_3$ [S2, unofficial].
    -   Positive faces flip U (`flip_u = true`) [S1, unofficial].
    -   If `flip_v` is true, V is flipped [S1, unofficial].
    -   UVs are baked with width/height so textures repeat [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_quad_geometry(face_dir: int, voxel: list, w: int, h: int, base_idx: int, flip_u: bool, flip_v: bool) -> tuple:
    # face_dir: 0=XNeg, 1=XPos, 2=YNeg, 3=YPos, 4=ZNeg, 5=ZPos
    # voxel: [x, y, z]
    
    # 1. Normal Calculation
    normals = {
        0: [-1, 0, 0], 1: [1, 0, 0],
        2: [0, -1, 0], 3: [0, 1, 0],
        4: [0, 0, -1], 5: [0, 0, 1]
    }
    normal = normals[face_dir]
    
    # 2. Axis Identification
    if face_dir in [0, 1]:
        normal_axis, u_axis, v_axis = 0, 2, 1
    elif face_dir in [2, 3]:
        normal_axis, u_axis, v_axis = 1, 2, 0
    else:
        normal_axis, u_axis, v_axis = 2, 0, 1
        
    # 3. Quad Position Generation
    origin = [float(voxel[0]), float(voxel[1]), float(voxel[2])]
    is_positive = face_dir in [1, 3, 5]
    if is_positive:
        origin[normal_axis] += 1.0
        
    v0 = origin[:]
    v1 = origin[:]
    v1[u_axis] += w
    v2 = origin[:]
    v2[v_axis] += h
    v3 = origin[:]
    v3[u_axis] += w
    v3[v_axis] += h
    
    local_verts = [v0, v1, v2, v3]
    
    # 4. Index Generation
    if face_dir in [0, 3, 5]: # XNeg, YPos, ZPos
        indices = [base_idx, base_idx+1, base_idx+2, base_idx+1, base_idx+3, base_idx+2]
    else: # XPos, YNeg, ZNeg
        indices = [base_idx, base_idx+2, base_idx+1, base_idx+1, base_idx+2, base_idx+3]
        
    # 5. UV Generation
    local_uvs = [(0.0, 0.0), (0.0, 1.0), (1.0, 1.0), (1.0, 0.0)]
    final_uvs = []
    for u, v in local_uvs:
        if is_positive:
            u = 1.0 - u
        if flip_u:
            u = 1.0 - u
        if flip_v:
            v = 1.0 - v
        final_uvs.append((u, v))
        
    return local_verts, final_uvs, indices, normal
```

**Edge Cases and Pitfalls:**
-   **Winding Order**: The winding order and axis conventions are chosen to match `block-mesh-rs`'s `RIGHT_HANDED_Y_UP_CONFIG` so that texture orientation and backface culling behavior remain consistent [S1, unofficial]. Incorrect winding will cause backface culling to hide visible faces.
-   **Positive Face Offset**: For positive faces, the origin must be offset by $1.0$ in the normal direction to align with the voxel grid boundary [S1, unofficial]. Failure to do so will result in geometry being placed inside the voxel rather than on its surface.
-   **UV Flipping**: Positive faces automatically flip U [S1, unofficial]. This must be accounted for when baking textures or using texture atlases.

## Subtopics

### [Static Index Buffer Generation](6255-static-index-buffer-generation.md)
Static Index Buffer Generation pre-generates index buffers for all possible Level of Detail (LOD) levels and stores them in GPU memory, allowing the renderer to bind the correct buffer directly during draw calls without per-frame regeneration or upload. This method exploits the high regularity of triangle topology in blocky meshing, where vertex connectivity depends only on grid resolution rather than elevation or density values. It differs from dynamic generation by eliminating CPU-side computation overhead per frame, trading initial memory allocation for rendering efficiency.

### [DCx Face Generation Priority Rules](9329-dcx-face-generation-priority-rules.md)
DCx Face Generation Priority Rules are a set of directional constraints used in Dual Contouring over Expanded Cubes (DCx) to prevent the generation of duplicate faces when adjacent expanded cubes overlap. They solve the ambiguity of shared face ownership by assigning responsibility for generating specific mesh faces based on the case ID of the current cube, the case ID of the neighbor, and a predefined directional priority. Unlike context-unaware methods, these rules ensure topological consistency across the boundary of overlapping expanded cubes without requiring post-processing duplicate removal.

### Voxel Face Normal and Offset Mapping
Maps an `EVoxelFace` enum to a unit normal vector and an integer offset vector for neighbor voxel access [S3, unofficial]. It also returns the opposite face direction for a given `EVoxelFace` [S3, unofficial].
-   **Normal Mapping**:
    -   Top: $(0, 0, 1)$
    -   Bottom: $(0, 0, -1)$
    -   North: $(0, 1, 0)$
    -   South: $(0, -1, 0)$
    -   East: $(1, 0, 0)$
    -   West: $(-1, 0, 0)$
    -   Default: ZeroVector
-   **Offset Mapping**:
    -   Top: $(0, 0, 1)$
    -   Bottom: $(0, 0, -1)$
    -   North: $(0, 1, 0)$
    -   South: $(0, -1, 0)$
    -   East: $(1, 0, 0)$
    -   West: $(-1, 0, 0)$
    -   Default: ZeroValue
-   **Opposite Mapping**:
    -   Top $\leftrightarrow$ Bottom
    -   North $\leftrightarrow$ South
    -   East $\leftrightarrow$ West
    -   Default: Face

## Operating characteristics
-   **Time Complexity**: Not given in the sources.
-   **Memory Complexity**: Not given in the sources.
-   **Measured Performance**: Not given in the sources.
-   **Scaling**: Scales linearly with the number of exposed faces.
-   **Strengths**: Simple, deterministic, and efficient for standard blocky rendering. Generates explicit vertex and index buffers for opaque and transparent meshes, rendered via standard draw calls [S2, unofficial].
-   **Limitations**: Generates a separate quad for each exposed face, which can lead to high vertex counts for complex terrain. Does not handle overlapping expanded cubes (requires DCx rules for that) [S1, unofficial].
-   **When to use**: Standard voxel rendering where simplicity and compatibility with standard pipelines are priorities.
-   **When not to use**: When overlapping expanded cubes are present or when memory bandwidth is a critical bottleneck and vertex sharing is required.

## Background
Originating from implementations such as `infinigen` [S1, unofficial], `VoxelEngine` [S2, unofficial], and `VoxelWorlds` [S3, unofficial], this method is a fundamental component of voxel face mesh generation. It provides the geometric foundation for rendering voxel chunks by defining how face orientations map to vertex positions and indices.

## Sources
[S1] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S2] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
[S3] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
