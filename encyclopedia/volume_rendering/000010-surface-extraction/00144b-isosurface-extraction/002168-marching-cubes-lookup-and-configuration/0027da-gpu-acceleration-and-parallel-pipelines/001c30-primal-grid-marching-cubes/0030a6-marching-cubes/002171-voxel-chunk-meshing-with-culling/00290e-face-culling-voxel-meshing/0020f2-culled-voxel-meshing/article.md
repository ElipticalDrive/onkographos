# Culled Voxel Meshing

Also called: Culled Meshing, Visible Face Generation with Boundary Policy

## Overview
Culled Voxel Meshing is a surface extraction method that generates a single mesh for a voxel chunk by omitting faces shared between adjacent solid voxels [S1, S2, unofficial]. It solves the problem of excessive geometry and draw calls inherent in naive per-voxel meshing by merging adjacent voxels into one large mesh [S1, unofficial]. It is distinguished from its parent [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md) by its explicit handling of chunk boundaries via a `BoundaryPolicy` parameter, and from [Greedy Meshing](3670-gpu-voxel-mesh-generation.md) by its lack of face merging (it produces more polygons but is simpler and faster) [S2, unofficial].

## How it works
**Inputs:** A voxel chunk data structure containing material IDs (where 0 represents air) and a `BoundaryPolicy` enum (`"air"` or `"solid"`).
**Outputs:** An array of `Quad` objects representing the visible surface.

**Data Structures:**
*   `VoxelChunk`: A 3D grid of size $S \times S \times S$. Access is linearized via index $i = x + y \cdot S + z \cdot S^2$. Out-of-bounds reads return 0 (air) by default, but the boundary policy overrides this for chunk edges [S2, unofficial].
*   `Quad`: Defined by axis, direction, slice position, and 2D coordinates $(u, v)$ with width/height (typically 1x1 for single voxels) and material ID [S2, unofficial].

**Algorithm Steps:**
1.  Initialize an empty list `quads`.
2.  Iterate through every voxel $(x, y, z)$ in the chunk.
3.  If the voxel material is 0 (air), skip it [S2, unofficial].
4.  For each of the 6 axes (X, Y, Z):
    a.  Determine the two directions along that axis: $+1$ and $-1$ [S2, unofficial].
    b.  For each direction $dir$:
        i.  Calculate the neighbor coordinates $(n_x, n_y, n_z)$ by adding $dir$ to the current coordinate on the current axis [S2, unofficial].
        ii. Determine if the neighbor is outside the chunk bounds [S2, unofficial].
        iii. Determine the neighbor's material value:
            *   If inside bounds: read from chunk data [S2, unofficial].
            *   If outside bounds: use `BoundaryPolicy`. If `"air"`, value is 0. If `"solid"`, value is non-zero (e.g., 1) [S2, unofficial].
        iv. If the neighbor's material is non-zero (solid), the face is internal and is **culled** (not emitted) [S2, unofficial].
        v. If the neighbor is air (or treated as air), emit a `Quad` for this face [S2, unofficial].
            *   The quad's axis is the current axis.
            *   The quad's direction is $dir$.
            *   The quad's slice position is the boundary between the current voxel and the neighbor (e.g., $z+1$ for direction $+1$ on Z) [S2, unofficial].
            *   The quad's 2D coordinates correspond to the current voxel's position on the other two axes [S2, unofficial].
            *   The material ID is the current voxel's material [S2, unofficial].
5.  Return `quads` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def meshCulled(chunk: VoxelChunk, policy: BoundaryPolicy) -> List[Quad]:
    quads = []
    s = chunk.size
    # AXES defines the 6 face directions: (axis_index, u_index, v_index, dir_sign)
    # e.g., (0, 1, 2, 1), (0, 1, 2, -1), etc.
    for x, y, z in chunk.voxels():
        material = chunk.get(x, y, z)
        if material == 0:
            continue
        
        for axis, u, v, dir_sign in AXES:
            # Calculate neighbor position
            nx, ny, nz = x, y, z
            if axis == 0: nx += dir_sign
            elif axis == 1: ny += dir_sign
            else: nz += dir_sign
            
            # Check bounds and determine neighbor material
            if nx < 0 or nx >= s or ny < 0 or ny >= s or nz < 0 or nz >= s:
                # Boundary policy
                if policy == "solid":
                    neighbor_material = 1 # Treat outside as solid
                else:
                    neighbor_material = 0 # Treat outside as air
            else:
                neighbor_material = chunk.get(nx, ny, nz)
            
            # Cull if neighbor is solid
            if neighbor_material != 0:
                continue
            
            # Emit quad
            # Slice is the coordinate of the face plane
            if axis == 0: slice_pos = x + dir_sign
            elif axis == 1: slice_pos = y + dir_sign
            else: slice_pos = z + dir_sign
            
            # 2D coords are the other two axes
            u_coord = ny if u == 1 else nz
            v_coord = nx if v == 0 else nz # Adjust based on axis mapping
            
            quads.append(makeQuad(axis, dir_sign, slice_pos, u_coord, v_coord, 1, 1, material))
            
    return quads
```

**Parameters:**
*   `BoundaryPolicy`: `"air"` (default for isolated chunks) or `"solid"` (for seamless chunking where outside is assumed solid) [S2, unofficial].

**Edge Cases:**
*   **Solid Chunk:** With `"air"` policy, a fully solid chunk produces only the outer shell. With `"solid"` policy, it produces 0 quads [S2, unofficial].
*   **Precision:** Standard integer arithmetic is sufficient for coordinates [S2, unofficial].

## Subtopics

### [GPU Voxel Mesh Generation](3670-gpu-voxel-mesh-generation.md)
A parallel variant optimized for GPU execution.

### [Face Visibility Check](3701-face-visibility-check.md)
A component method used to determine if a specific face is visible before meshing.

### [Voxel Visibility Map Computation](4046-voxel-visibility-map-computation.md)
A pre-computation step to identify visible voxels.

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is visited once and 6 neighbors are checked [S2, unofficial].
**Memory Complexity:** $O(F)$ where $F$ is the number of visible faces, for the output quad array [S2, unofficial].
**Performance:** On an Apple M2 Pro, culled meshing for a 32³ terrain chunk (17,191 solid voxels) takes a median of 2.06 ms [S2, unofficial]. It is significantly faster than naive meshing (5.60 ms) but slower than greedy meshing due to the lack of face merging [S2, unofficial].
**Geometry Reduction:** Reduces polygon count compared to naive meshing (6,654 quads vs 103,146 quads for the same terrain) but produces more geometry than greedy meshing (6,654 vs 2,416 quads) [S2, unofficial].
**Strengths:** Simple implementation, deterministic, low latency, no complex data structures [S2, unofficial].
**Limitations:** Still produces more polygons than greedy meshing; does not merge adjacent coplanar faces [S1, S2, unofficial].

## Background
Culled meshing is a standard technique in voxel game engines, notably featured in the NOA voxel game engine [S1, unofficial]. It is often compared to naive and greedy meshing as part of a spectrum of voxel surface extraction methods [S2, unofficial]. The specific implementation details and benchmarks described here originate from a TypeScript implementation and accompanying article by m2-md [S2, unofficial].

## Sources
[S1] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S2] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
