# Greedy Meshing for Voxel Faces

Also called: Greedy Meshing with Face Attributes, Greedy meshing for complex voxel data, Voxel face greedy meshing

## Overview
Greedy meshing for voxel faces extends standard greedy meshing by merging adjacent coplanar voxel faces based on multiple attributes (such as material type and transparency) rather than simple voxel existence [S1, unofficial][S2, unofficial]. Unlike basic greedy meshing which merges voxels of the same type, this method uses a custom equality check on face objects to allow for complex culling and attribute-based optimization [S1, unofficial][S2, unofficial].

## How it works
The algorithm processes a voxel chunk by iterating through six face directions (front and back for each of the three axes) [S2, unofficial]. For each direction, it generates a 2D mask of visible faces and then greedily merges adjacent identical faces into larger quads [S1, unofficial][S2, unofficial].

### Inputs and Outputs
*   **Input**: A 3D array of `VoxelFace` objects representing the voxel chunk. The array dimensions are `CHUNK_WIDTH`, `CHUNK_HEIGHT`, and `CHUNK_DEPTH` (assumed equal to `CHUNK_WIDTH` in the reference implementation) [S1, unofficial][S2, unofficial].
*   **Output**: A set of quads, each defined by position, size, and the `VoxelFace` attributes [S1, unofficial][S2, unofficial].

### Data Structures
**VoxelFace**
Encapsulates the attributes of a single voxel face. The `equals` method determines mergeability [S1, unofficial].
*   `transparent` (boolean): If true, the face is culled and skipped during meshing [S1, unofficial][S2, unofficial].
*   `type` (int): The material or voxel type identifier [S1, unofficial][S2, unofficial].
*   `side` (int): The face direction constant [S1, unofficial][S2, unofficial].
    *   `SOUTH = 0`
    *   `NORTH = 1`
    *   `EAST = 2`
    *   `WEST = 3`
    *   `TOP = 4`
    *   `BOTTOM = 5`
*   `equals(VoxelFace face)`: Returns `true` if `face.transparent == this.transparent && face.type == this.type` [S1, unofficial][S2, unofficial].

**Working Arrays**
*   `x[3]`: Current position in the chunk (indices 0, 1, 2).
*   `q[3]`: Direction vector for neighbor checks (one element is 1, others 0).
*   `du[3]`: Delta vector for the quad's width dimension (one element is width `w`, others 0).
*   `dv[3]`: Delta vector for the quad's height dimension (one element is height `h`, others 0).
*   `mask`: A 1D array of size `CHUNK_WIDTH * CHUNK_HEIGHT` storing `VoxelFace` references or `null` [S1, unofficial][S2, unofficial].

### Parameters
*   `VOXEL_SIZE`: Size of a single voxel in world units. Default is 1. Used to scale vertex coordinates [S1, unofficial][S2, unofficial].
*   `CHUNK_WIDTH`, `CHUNK_HEIGHT`: Dimensions of the voxel chunk. In the reference example, both are 3 [S2, unofficial].

### Algorithm Steps
1.  **Initialize Working Variables**: Initialize `x = [0, 0, 0]`, `q = [0, 0, 0]`, `du = [0, 0, 0]`, `dv = [0, 0, 0]`. Allocate `mask` array of size `CHUNK_WIDTH * CHUNK_HEIGHT` [S1, unofficial].
2.  **Iterate Face Directions**: Use a boolean toggle loop to handle both back-face and front-face culling/directions: `backFace = True; b = False; while b != backFace: ... backFace = backFace and b; b = not b`. This loop runs twice: once for back faces, once for front faces [S1, unofficial].
3.  **Process Each Dimension**: Inside the direction loop, iterate `d` from 0 to 2 (representing X, Y, Z axes) [S2, unofficial]. Calculate perpendicular axes: `u = (d + 1) % 3`, `v = (d + 2) % 3` [S2, unofficial]. Reset `x` to `[0, 0, 0]`. Set `q[d] = 1` (direction vector for neighbor check), others 0 [S2, unofficial]. Determine `side` (face normal) based on `d` and `backFace`: If `d == 0`: `side = WEST` if `backFace` else `EAST` [S2, unofficial]. If `d == 1`: `side = BOTTOM` if `backFace` else `TOP` [S2, unofficial]. If `d == 2`: `side = SOUTH` if `backFace` else `NORTH` [S2, unofficial].
4.  **Sweep and Generate Mask**: Sweep along the depth dimension `d` from `x[d] = -1` to `x[d] < CHUNK_WIDTH` [S1, unofficial]. For each `x[d]`: Reset `n = 0` [S1, unofficial]. Iterate `x[v]` from 0 to `CHUNK_HEIGHT - 1` [S2, unofficial]. Iterate `x[u]` from 0 to `CHUNK_WIDTH - 1` [S2, unofficial]. Get `voxelFace` at `(x[0], x[1], x[2])` if `x[d] >= 0`, else `null` [S2, unofficial]. Get `voxelFace1` at `(x[0] + q[0], x[1] + q[1], x[2] + q[2])` if `x[d] < CHUNK_WIDTH - 1`, else `null` [S2, unofficial]. If both faces exist and `voxelFace.equals(voxelFace1)` is true, set `mask[n] = null` (faces are internal/identical, so no boundary) [S1, unofficial]. Otherwise, set `mask[n] = voxelFace1` if `backFace` is true, else `mask[n] = voxelFace` [S1, unofficial]. Increment `n`. Increment `x[d]` [S1, unofficial].
5.  **Greedy Mesh Generation**: Reset `n = 0` [S1, unofficial]. Iterate `j` from 0 to `CHUNK_HEIGHT - 1` [S1, unofficial]. Iterate `i` from 0 to `CHUNK_WIDTH - 1` (note: `i` is incremented by `w` inside the loop, so use a `while` or conditional increment) [S1, unofficial]. If `mask[n]` is not `null`: Compute Width `w`: Start `w = 1`. While `i + w < CHUNK_WIDTH` and `mask[n + w]` is not `null` and `mask[n + w].equals(mask[n])`, increment `w`. Compute Height `h`: Start `h = 1`. Set `done = false`. While `j + h < CHUNK_HEIGHT` and not `done`: Loop `k` from 0 to `w - 1`: If `mask[n + k + h * CHUNK_WIDTH]` is `null` or `!mask[n + k + h * CHUNK_WIDTH].equals(mask[n])`: Set `done = true` and break inner loop. If not `done`, increment `h`. Create Quad: If `mask[n].transparent` is `false`: Set `x[u] = i`, `x[v] = j`. Set `du[u] = w`, `du` others 0. Set `dv[v] = h`, `dv` others 0. Call `quad()` with vertices: `P0 = (x[0], x[1], x[2])`, `P1 = (x[0] + du[0], x[1] + du[1], x[2] + du[2])`, `P2 = (x[0] + du[0] + dv[0], x[1] + du[1] + dv[1], x[2] + du[2] + dv[2])`, `P3 = (x[0] + dv[0], x[1] + dv[1], x[2] + dv[2])`. Pass `width=w`, `height=h`, `face=mask[n]`, and `backFace` flag to the renderer. Zero Out Processed Mask: Loop `l` from 0 to `h - 1`. Loop `k` from 0 to `w - 1`. Set `mask[n + k + l * CHUNK_WIDTH] = null`. Advance `i` by `w` and `n` by `w` to continue scanning the current row [S1, unofficial]. If `mask[n]` is `null`, advance `i` and `n` by 1 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_faces(voxels, chunk_width, chunk_height, voxel_size):
    SOUTH, NORTH, EAST, WEST, TOP, BOTTOM = 0, 1, 2, 3, 4, 5
    quads = []
    
    x = [0, 0, 0]
    q = [0, 0, 0]
    mask = [None] * (chunk_width * chunk_height)
    
    backFace = True
    b = False
    while b != backFace:
        for d in range(3):
            u = (d + 1) % 3
            v = (d + 2) % 3
            x[0] = x[1] = x[2] = 0
            q[0] = q[1] = q[2] = 0
            q[d] = 1
            
            if d == 0:
                side = WEST if backFace else EAST
            elif d == 1:
                side = BOTTOM if backFace else TOP
            else:
                side = SOUTH if backFace else NORTH
                
            for x_d in range(-1, chunk_width):
                x[d] = x_d
                n = 0
                for x_v in range(chunk_height):
                    for x_u in range(chunk_width):
                        x[u] = x_u
                        x[v] = x_v
                        
                        # Retrieve faces with bounds checking [G]
                        voxelFace = voxels[x[0]][x[1]][x[2]] if 0 <= x[0] < chunk_width and 0 <= x[1] < chunk_height and 0 <= x[2] < chunk_width and x[d] >= 0 else None
                        voxelFace1 = voxels[x[0] + q[0]][x[1] + q[1]][x[2] + q[2]] if 0 <= x[0] + q[0] < chunk_width and 0 <= x[1] + q[1] < chunk_height and 0 <= x[2] + q[2] < chunk_width and x[d] < chunk_width - 1 else None
                        
                        if voxelFace is not None and voxelFace1 is not None and voxelFace.equals(voxelFace1):
                            mask[n] = None
                        else:
                            mask[n] = voxelFace1 if backFace else voxelFace
                        n += 1
                        
                n = 0
                j = 0
                while j < chunk_height:
                    i = 0
                    while i < chunk_width:
                        if mask[n] is not None:
                            w = 1
                            while i + w < chunk_width and mask[n + w] is not None and mask[n + w].equals(mask[n]):
                                w += 1
                            
                            h = 1
                            done = False
                            while j + h < chunk_height and not done:
                                for k in range(w):
                                    if mask[n + k + h * chunk_width] is None or not mask[n + k + h * chunk_width].equals(mask[n]):
                                        done = True
                                        break
                                if not done:
                                    h += 1
                            
                            if not mask[n].transparent:
                                du = [0, 0, 0]
                                dv = [0, 0, 0]
                                du[u] = w
                                dv[v] = h
                                
                                p0 = [x[0] * voxel_size, x[1] * voxel_size, x[2] * voxel_size]
                                p1 = [p0[0] + du[0] * voxel_size, p0[1] + du[1] * voxel_size, p0[2] + du[2] * voxel_size]
                                p2 = [p1[0] + dv[0] * voxel_size, p1[1] + dv[1] * voxel_size, p1[2] + dv[2] * voxel_size]
                                p3 = [p0[0] + dv[0] * voxel_size, p0[1] + dv[1] * voxel_size, p0[2] + dv[2] * voxel_size]
                                quads.append({'vertices': [p0, p1, p2, p3], 'face': mask[n]})
                            
                            for l in range(h):
                                for k in range(w):
                                    mask[n + k + l * chunk_width] = None
                            
                            i += w
                            n += w
                        else:
                            i += 1
                            n += 1
                    j += 1
        backFace = backFace and b
        b = not b
    return quads
```

### Edge Cases and Pitfalls
*   **Transparent Faces**: Transparent faces must be skipped during meshing to avoid rendering culled geometry. The algorithm checks `mask[n].transparent` before adding a quad [S1, unofficial][S2, unofficial].
*   **Boundary Conditions**: The sweep loop for `x[d]` starts at -1 to handle the boundary between the chunk and its neighbors. Neighbor checks must handle indices outside the chunk bounds by returning `null` [S1, unofficial][S2, unofficial].
*   **Precision**: Vertex coordinates are scaled by `VOXEL_SIZE` to match world units [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The algorithm processes each face direction independently. For each direction, it sweeps through the chunk dimensions. The greedy merging step is linear with respect to the number of faces in the mask, as each face is visited and zeroed out once. Overall complexity is proportional to the surface area of the voxel chunk times the number of dimensions (constant factor).
*   **Memory Complexity**: Requires a mask array of size `CHUNK_WIDTH * CHUNK_HEIGHT` per face direction. Working arrays are constant size.
*   **Performance**: Reduces triangle count by merging adjacent coplanar faces. The reduction factor depends on the homogeneity of the voxel data.
*   **Strengths**: Handles complex face attributes (type, transparency, lighting) allowing for more accurate culling and merging than simple voxel-type-based greedy meshing.
*   **Limitations**: Requires a custom `equals` method for `VoxelFace`. Performance may degrade if faces have many unique attributes, reducing merge opportunities.

## Background
Greedy meshing for voxel faces was introduced by Robert O'Leary as an extension of Mikola Lysenko's original greedy meshing algorithm [S1, unofficial][S2, unofficial]. Lysenko's original algorithm merged voxels based on type alone. O'Leary's modification allows for comparison based on multiple attributes simultaneously, making it suitable for complex voxel engines with varying face properties like sunlight, artificial light, and material types [S1, unofficial][S2, unofficial]. The implementation was shared on the jMonkeyEngine Hub in December 2013 [S2, unofficial].

## Sources
[S1] roboleary/GreedyMesh, https://github.com/roboleary/GreedyMesh, fetched 2026-10-26 (unofficial)
[S2] Greedy meshing for complex voxel data, https://hub.jmonkeyengine.org/t/greedy-meshing-for-complex-voxel-data/28017, fetched 2026-10-27 (unofficial)
