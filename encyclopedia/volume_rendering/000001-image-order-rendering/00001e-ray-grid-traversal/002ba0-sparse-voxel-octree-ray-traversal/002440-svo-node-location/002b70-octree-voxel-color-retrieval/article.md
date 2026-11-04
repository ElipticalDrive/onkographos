# Octree Voxel Color Retrieval

Also called: Color Lookup with Shading, lookupColor

## Overview
Retrieves the color of a voxel from the sparse voxel octree (SVO) data structure, applying lighting if the scene is not shadeless [S1, unofficial]. It navigates the tree from a given node to find the leaf node containing the color data [S1, unofficial].

## How it works
**Inputs:**
- `castRes`: Result from ray traversal containing `pos` (float3), `node` (uint), `childIdx` (uint), `scale` (uint), and `normal` (float3) [S1, unofficial].
- `octree`: Data structure containing a flat `data` array of uints [S1, unofficial].
- `renderList`: Contains an `offsets` array for instances [S1, unofficial].
- `ubo`: Uniform buffer object containing `shadeless` (int), `ambientStrength` (scalar), `lightColor` (vec3), `lightPos` (vec3), `blockInfoEnd` (offset), `pageBytes` (bytes), and `transformCount` (int) [S1, unofficial].
- `index`: Instance index [S1, unofficial].

**Outputs:**
- `vec4`: Final color with alpha [S1, unofficial].

**Data Structures:**
The octree is stored in a flat array `octree.data` [S1, unofficial]. Nodes are organized into pages with headers, and the layout relies on bit-packing for node pointers and color data [S1, unofficial].

**Steps:**

1. **Extract Position and Tree State:**
   Convert ray position to integer coordinates for indexing.
   $$ px = \text{floatBitsToInt}(castRes.pos.x) $$
   $$ node = castRes.node $$
   $$ cidx = castRes.childIdx $$
   $$ level = castRes.scale $$
   [S1, unofficial]

2. **Calculate Page Header and Block Info:**
   Extract the page header from the node index.
   $$ pageHeader = node \ \& \ 0xFFFFF800 $$
   Retrieve block information from the data array.
   $$ blockInfo = pageHeader + octree.data[pageHeader] $$
   Get attach data by adding the block info end offset.
   $$ attachData = \text{int}(blockInfo) + ubo.blockInfoEnd $$
   [S1, unofficial]

3. **Get Palette Node:**
   Calculate the palette node index.
   $$ paletteNode = octree.data[attachData + \text{int}(node) - \text{renderList.offsets}[index] / 4 - 1] $$
   [S1, unofficial]

4. **Traverse to Leaf:**
   Loop while the current node does not have color data (indicated by the bit at `cidx` not being set).
   $$ \text{while } ((\text{int}(paletteNode \gg cidx) \ \& \ 1) \neq 1) $$
   Inside the loop: increment level, check bounds, and update `node`, `cidx`, `pageHeader`, `blockInfo`, `attachData`, and `paletteNode` according to the octree traversal logic [S1, unofficial].

5. **Calculate Attach Offset and Extract Color Component:**
   $$ attachOffset = attachData + \text{int}(paletteNode \gg 8) + \text{bitCount}(paletteNode \ \& \ (1 \ll cidx) - 1) $$
   $$ c = \text{int}(octree.data[attachOffset]) $$
   [S1, unofficial]

6. **Extract RGB and Normalize:**
   $$ r = (c \gg 16) \ \& \ 0xFF $$
   $$ g = (c \gg 8) \ \& \ 0xFF $$
   $$ b = c \ \& \ 0xFF $$
   $$ octreeColor = \text{vec3}(r / d, g / d, b / d) $$
   where $d$ is a divisor used for normalization [S1, unofficial].

7. **Handle Shadeless Mode:**
   If `ubo.shadeless == 1`, return the color with alpha 1.0.
   $$ \text{return vec4}(octreeColor, 1.0) $$
   [S1, unofficial]

8. **Load Transformation Matrix:**
   Calculate offset for transform data.
   $$ offset = \text{int}(\text{renderList.offsets}[index] + ubo.pageBytes) / 4 - ubo.transformCount * 4 $$
   Read 16 floats into `v[]` and construct `mat4 octreeToWorld` [S1, unofficial].

9. **Normalize Hit Normal in World Space:**
   $$ hitNormalWorld = \text{normalize}(octreeToWorld * castRes.normal) $$
   [S1, unofficial]

10. **Calculate Lighting:**
    $$ ambient = ubo.ambientStrength * \text{vec3}(ubo.lightColor) $$
    $$ lightDir = \text{vec3}(\text{normalize}(ubo.lightPos)) $$
    $$ diffuse = \max(\text{dot}(hitNormalWorld, lightDir), 0.0) * \text{vec3}(ubo.lightColor) $$
    $$ color = (ambient + diffuse) * octreeColor $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def float_bits_to_int(f):
    return struct.unpack('i', struct.pack('f', f))[0]

def bit_count(n):
    return bin(n).count('1')

# Standard vector/matrix operations assumed available
def vec3(x, y, z): return (x, y, z)
def vec4(x, y, z, w): return (x, y, z, w)
def mat4(v): return [v[i:i+4] for i in range(0, 16, 4)]
def normalize(v):
    l = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return (v[0]/l, v[1]/l, v[2]/l) if l else (0,0,0)
def dot(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def max(a, b): return a if a > b else b

def octree_voxel_color_retrieval(castRes, octree, renderList, ubo, index):
    # 1. Extract Position and Tree State
    px = float_bits_to_int(castRes.pos.x)
    node = castRes.node
    cidx = castRes.childIdx
    level = castRes.scale

    # 2. Calculate Page Header and Block Info
    page_header = node & 0xFFFFF800
    block_info = page_header + octree.data[page_header]
    attach_data = int(block_info) + ubo.block_info_end

    # 3. Get Palette Node
    palette_node = octree.data[attach_data + int(node) - renderList.offsets[index] // 4 - 1]

    # 4. Traverse to Leaf
    while (int(palette_node >> cidx) & 1) != 1:
        level += 1
        # [G] Check bounds and update node, cidx, pageHeader, blockInfo, attachData, paletteNode
        # according to standard SVO traversal logic. Sources do not specify the exact update formulas.
        pass

    # 5. Calculate Attach Offset and Extract Color Component
    attach_offset = attach_data + int(palette_node >> 8) + bit_count(palette_node & ((1 << cidx) - 1))
    c = int(octree.data[attach_offset])

    # 6. Extract RGB and Normalize
    r = (c >> 16) & 0xFF
    g = (c >> 8) & 0xFF
    b = c & 0xFF
    octree_color = vec3(r / d, g / d, b / d)

    # 7. Handle Shadeless Mode
    if ubo.shadeless == 1:
        return vec4(octree_color, 1.0)

    # 8. Load Transformation Matrix
    offset = int(renderList.offsets[index] + ubo.page_bytes) // 4 - ubo.transform_count * 4
    v = [octree.data[offset + i] for i in range(16)]
    octree_to_world = mat4(v)

    # 9. Normalize Hit Normal in World Space
    hit_normal_world = normalize(octree_to_world * castRes.normal)

    # 10. Calculate Lighting
    ambient = ubo.ambient_strength * vec3(ubo.light_color)
    light_dir = vec3(normalize(ubo.light_pos))
    diffuse = max(dot(hit_normal_world, light_dir), 0.0) * vec3(ubo.light_color)
    color = (ambient + diffuse) * octree_color

    return vec4(color, 1.0)
```

## Operating characteristics
Time complexity is linear with respect to the octree depth from the hit node to the leaf [S1, unofficial]. Memory complexity is constant for local variables [G]. Not given in the sources: specific performance metrics, scaling behavior, or explicit usage recommendations.

## Background
Derived from the `origin` repository, which implements sparse voxel octree rendering [S1, unofficial]. Used in real-time ray tracing to provide shaded voxel colors [S1, unofficial].

## Sources
[S1] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
