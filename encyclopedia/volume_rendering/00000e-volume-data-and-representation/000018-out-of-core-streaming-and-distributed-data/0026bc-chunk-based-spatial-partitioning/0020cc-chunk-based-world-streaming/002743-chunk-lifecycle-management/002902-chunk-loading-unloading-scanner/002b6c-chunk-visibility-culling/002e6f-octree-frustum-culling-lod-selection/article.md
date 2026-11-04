# Octree Frustum Culling & LOD Selection

Also called: CPU Frustum Culling, CPU-based Octree LOD & Frustum Culling

## Overview
This method recursively traverses a chunk-based octree to determine which spatial regions are visible within the camera's view frustum and assigns an appropriate Level of Detail (LOD) based on the distance from the camera. It differs from sibling techniques by combining visibility culling and LOD selection into a single recursive traversal, deferring geometry queueing until a node is determined to be either fully visible at its current LOD or fully outside the frustum. It builds on [Chunk Visibility Culling](5031-chunk-visibility-culling.md) by adding distance-based LOD logic and explicit frustum plane intersection tests.

## How it works
**Inputs:**
- `set`: The chunk set structure containing octree metadata, root size, and geometry arrays.
- `cam`: Camera structure containing `frustum_planes` (array of 6 planes) and `lod_origin` (camera position).
- `lod`: Current level of detail (unsigned 8-bit integer).
- `x, y, z`: Chunk coordinates at the current `lod` level.

**Outputs:**
- Populated geometry queues: `queue_gsvl`, `queue_mesh`, `queue_misc`.

**Data Structures:**
The chunk set (`set`) contains:
- `root`: Base root size of the chunk set.
- `max[3]`: Maximum chunk coordinates for each axis.
- `max_bitw[3]`: Bit width required to index the maximum coordinate for each axis.
- `gsvl[lod][index]`: Array of `GeometrySVL*` pointers for structured volume geometry.
- `gmesh[index]`: Array of `GeometryMesh*` pointers for mesh geometry.

The queue structures contain:
- `queue_gsvl[0xFFFF]`: Array of `GeometrySVL*` pointers.
- `queue_mesh[0xFFFF]`: Array of `GeometryMesh*` pointers.
- `queue_misc[0xFFFF]`: Array of `uint32_t` indices.
- `queue_gsvl_count`, `queue_mesh_count`, `queue_misc_count`: Current counts of enqueued items.

**Parameters:**
- `ws_root`: World space root size, calculated as `set->root << lod`.
- `LOD_SCALE`: Distance scale factor, value `1024`.
- `FRUSTUM_RADIUS_MULT`: Frustum culling radius multiplier, value `0.85`.

**Algorithm Steps:**
1. **Compute World-Space Center:** Calculate the center of the current chunk in world space.
   $$ center[i] = (coord[i] \times ws\_root) + \frac{ws\_root}{2} $$
   where $i \in \{0, 1, 2\}$ corresponds to x, y, z axes.

2. **Boundary Check:** Verify the chunk coordinates are within the defined bounds of the chunk set.
   $$ \text{if } (coord[i] \ll lod) \ge set.max[i] \text{ for any } i, \text{ return.} $$

3. **Frustum Culling:** Check if the chunk intersects the camera frustum. The chunk is treated as a sphere or box with a radius scaled by `0.85` of the world-space root size.
   $$ \text{if } \neg \text{gfx\_fcull\_visible}(cam.frustum\_planes, center, ws\_root \times 0.85), \text{ return.} $$
   [G] `gfx_fcull_visible` performs a standard sphere/box vs. 6-plane frustum intersection test. If the object is completely outside any plane, it returns false.

4. **Distance Calculation:** Compute the Euclidean distance from the camera's LOD origin to the chunk's world-space center.
   $$ distance = \sqrt{\sum_{i=0}^{2} (center[i] - cam.lod\_origin[i])^2} $$

5. **LOD Decision:**
   - **If** $distance > lod \times 1024$: The chunk is far enough to be rendered at the current LOD. Enqueue the appropriate geometry and return.
   - **Else**: The chunk is close enough to require a higher resolution (lower LOD index). Decrement `lod` and recurse into the 8 children.

6. **Geometry Enqueuing (Far Case):**
   - Calculate the bit widths for the flattened index:
     $$ octree\_bitw[i] = set.max\_bitw[i] - \min(lod, set.max\_bitw[i]) $$
   - **If** $lod == 0$:
     - Calculate the mesh index: $index = \text{flatten3}((uint16_t[])\{x, y, z\}, set.max\_bitw)$.
     - Retrieve $m = set.gmesh[index]$.
     - If $m.vbo$ exists, enqueue $m$ into `queue_mesh` and $index$ into `queue_misc`.
     - Return.
   - **Else**:
     - Calculate the SVL index: $idx = \text{flatten3}((uint16_t[])\{x, y, z\}, octree\_bitw)$.
     - Retrieve $g = set.gsvl[lod][idx]$.
     - If $g.vbo$ exists, enqueue $g$ into `queue_gsvl`.
     - Return.

7. **Recursion (Close Case):**
   - Decrement `lod`.
   - Iterate $nx, ny, nz \in \{0, 1\}$.
   - Recursively call the algorithm with coordinates $((x \times 2) + nx, (y \times 2) + ny, (z \times 2) + nz)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octree_cull(set, cam, lod, x, y, z):
    # 1. Compute World-Space Root Size
    ws_root = set.root << lod
    
    # 2. Boundary Check
    for i in range(3):
        coord = [x, y, z][i]
        if (coord << lod) >= set.max[i]:
            return

    # 3. Compute World-Space Center
    center = [0.0, 0.0, 0.0]
    for i in range(3):
        coord = [x, y, z][i]
        center[i] = (coord * ws_root) + (ws_root / 2.0)

    # 4. Frustum Culling
    # gfx_fcull_visible checks if the sphere at 'center' with radius 'ws_root * 0.85'
    # intersects any of the 6 planes in cam.frustum_planes.
    if not gfx_fcull_visible(cam.frustum_planes, center, ws_root * 0.85):
        return

    # 5. Distance Calculation
    distance = glm_vec_distance(cam.lod_origin, center)

    # 6. LOD Decision
    if distance > lod * 1024:
        # Enqueue geometry for current LOD
        if lod == 0:
            # Mesh Geometry
            index = flatten3([x, y, z], set.max_bitw)
            m = set.gmesh[index]
            if m.vbo:
                queue_mesh[queue_mesh_count] = m
                queue_mesh_count += 1
                queue_misc[queue_misc_count] = index
                queue_misc_count += 1
        else:
            # SVL Geometry
            octree_bitw = [0, 0, 0]
            for i in range(3):
                octree_bitw[i] = set.max_bitw[i] - min(lod, set.max_bitw[i])
            
            idx = flatten3([x, y, z], octree_bitw)
            g = set.gsvl[lod][idx]
            if g.vbo:
                queue_gsvl[queue_gsvl_count] = g
                queue_gsvl_count += 1
        return

    # 7. Recurse into children
    new_lod = lod - 1
    for nx in range(2):
        for ny in range(2):
            for nz in range(2):
                octree_cull(set, cam, new_lod, (x * 2) + nx, (y * 2) + ny, (z * 2) + nz)

def flatten3(coords, bitw):
    # Flattens 3D coordinates into a 1D index using provided bit widths
    # Assumes little-endian or specific packing order consistent with the source
    # Implementation detail: typically (z << (bitw[0]+bitw[1])) | (y << bitw[0]) | x
    # Exact packing order depends on 'set' implementation, but standard is XYZ or ZYX.
    # Based on source usage, it treats coords as uint16_t array.
    x, y, z = coords
    b0, b1, b2 = bitw
    # Note: Source does not specify exact bit shift order, but standard voxel hashing is often ZYX or XYZ.
    # Assuming standard interleaved bit packing or simple shift.
    # Given 'max_bitw' is used for mesh and 'octree_bitw' for SVL, the packing must be consistent.
    # A common safe assumption for 'flatten3' in such contexts:
    return (z << (b0 + b1)) | (y << b0) | x
```

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of nodes in the octree, but typically much less due to frustum culling skipping large portions of the tree. The traversal is depth-first.
- **Memory Complexity:** $O(D)$ where $D$ is the maximum depth of the octree (recursion stack). The queue sizes are bounded by $0xFFFF$ entries per type.
- **Performance:** Efficient for large sparse volumes as it avoids processing invisible or distant low-detail chunks. The fixed-size queues prevent memory fragmentation but may limit the number of chunks processed per frame if the scene is extremely dense.
- **Strengths:** Simple implementation, low CPU overhead per frame, effective culling.
- **Limitations:** CPU-bound traversal can become a bottleneck for very deep octrees or high frame rates. The fixed LOD scale factor (1024) may not suit all scene scales.

## Background
This method is derived from the voxel platforming engine implementation in the `voxplat` repository by kosshi-net [S1, unofficial]. It represents a practical approach to managing chunk-based voxel data in real-time applications, balancing visibility culling with LOD selection to maintain performance.

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
