# Octree Voxel Color Retrieval

Also called: Color Lookup with Shading, Voxel Lighting and Color Retrieval, lookupColor

## Overview
Octree Voxel Color Retrieval computes the final color of a hit voxel by retrieving its stored base color and normal data from a Sparse Voxel Octree (SVO) and applying a directional lighting model [S1, S2, unofficial]. It differs from [Octree Node Lookup by Voxel](5120-octree-node-lookup-by-voxel.md) by explicitly handling the shading calculation (ambient and diffuse) and the specific bit-packing formats used to store color and normals in the leaf nodes, rather than just returning raw geometric or density data [S1, S2, unofficial].

## How it works
**Inputs:**
- Ray hit position, normal, and tree state from [SVO Node Location](3249-svo-node-location.md) [S2, unofficial].
- Global lighting parameters: light direction vector $L$, ambient strength, light color [S1, S2, unofficial].
- Octree data structure containing packed color/normal data and transformation matrices [S1, S2, unofficial].

**Outputs:**
- Final 3D color vector (RGB) and alpha value [S1, S2, unofficial].

**Data Structures:**
- **Voxel Color/Normal Storage:** Stored as 32-bit integers (uint32_t) in the octree data array [S1, S2, unofficial].
  - **Format A (S1):** RGB components are packed into the lower 24 bits. The normal is stored separately or implied [S1, unofficial].
  - **Format B (S2):** A single 32-bit integer contains RGB components packed as `(R << 16) | (G << 8) | B`, where R, G, B are 8-bit integers [S2, unofficial]. The normal is stored as a separate 3D vector, often normalized or stored as 8-bit integers divided by 128 [S1, S2, unofficial].
- **Tree Navigation State:**
  - `node`: Current node index [S2, unofficial].
  - `cidx`: Child index within the current node [S2, unofficial].
  - `level`: Current octree level [S2, unofficial].
  - `pageHeader`: Page-aligned address for block info lookup [S2, unofficial].
  - `blockInfo`: Metadata for the current block [S2, unofficial].
  - `attachData`: Pointer to attach data [S2, unofficial].
  - `paletteNode`: Reference to the color/normal palette entry [S2, unofficial].

**Steps:**

1. **Extract Hit State:** Retrieve position, node index, child index, and level from the ray traversal result [S2, unofficial].
   ```python
   px = floatBitsToInt(hit.pos.x)
   node = hit.node
   cidx = hit.childIdx
   level = hit.scale
   ```

2. **Navigate to Color Data:** Traverse up the tree to find the leaf node containing the color data [S2, unofficial].
   - Calculate `pageHeader = node & 0xFFFFF800` [S2, unofficial].
   - Get `blockInfo = pageHeader + octree.data[pageHeader]` [S2, unofficial].
   - Get `attachData = int(blockInfo) + ubo.blockInfoEnd` [S2, unofficial].
   - Get `paletteNode = octree.data[attachData + int(node) - renderList.offsets[index] / 4 - 1]` [S2, unofficial].
   - Loop while the specific bit corresponding to `cidx` in `paletteNode` is not set (indicating no color data at this level) [S2, unofficial]:
     - Increment `level`.
     - Check bounds.
     - Update `node`, `cidx`, `pageHeader`, `blockInfo`, `attachData`, and `paletteNode` according to the SVO hierarchy rules [S2, unofficial].

3. **Retrieve Base Color:**
   - Calculate `attachOffset = attachData + int(paletteNode >> 8) + bitCount(paletteNode & (1 << cidx) - 1)` [S2, unofficial].
   - Read color component `c = octree.data[attachOffset]` [S2, unofficial].
   - Extract RGB: `r = (c >> 16) & 0xFF`, `g = (c >> 8) & 0xFF`, `b = c & 0xFF` [S2, unofficial].
   - Normalize to [0, 1]: `octreeColor = vec3(r / d, g / d, b / d)`, where `d` is the normalization divisor (typically 255.0) [S2, unofficial].

4. **Apply Lighting:**
   - **If Shadeless:** Return `vec4(octreeColor, 1.0)` [S2, unofficial].
   - **If Shaded:**
     - Load transformation matrix `octreeToWorld` from the render list [S2, unofficial].
     - Transform hit normal to world space: `hitNormalWorld = normalize(octreeToWorld * hit.normal)` [S2, unofficial].
     - Calculate Ambient: `ambient = ubo.ambientStrength * vec3(ubo.lightColor)` [S2, unofficial].
     - Calculate Diffuse: `diffuse = max(dot(vec3(hitNormalWorld), lightDir), 0.0) * vec3(ubo.lightColor)` [S2, unofficial].
     - Final Color: `color = (ambient + diffuse) * octreeColor` [S2, unofficial].

**Alternative Formula (S1):**
Some implementations compute brightness directly from the stored normal $N$ (8-bit integers divided by 128) and light direction $L$ [S1, unofficial]:
$$ \text{brightness} = \frac{\arccos\left( \frac{N_x}{128.0} L_x + \frac{N_y}{128.0} L_y + \frac{N_z}{128.0} L_z \right)}{\pi} $$
Final color components are then:
$$ R = G = \text{unsigned int}(V_{\text{color}}[1] \times \text{brightness}) $$
$$ B = \text{unsigned int}(V_{\text{color}}[0] \times \text{brightness}) $$
Packed as `(R << 16) | (G << 8) | B` [S1, unofficial].

**Parameters:**
- **Light Direction (S1):** `{-0.039030f, -0.866792f, 0.497139f}` [S1, unofficial].
- **Normalization Divisor (S2):** Not explicitly given, but implied by 8-bit RGB extraction [S2, unofficial].

**Edge Cases:**
- **Shadeless Mode:** Bypasses lighting calculation entirely [S2, unofficial].
- **Missing Color Data:** The loop in Step 2 handles cases where color data is not present at the current leaf level, requiring traversal to a parent or sibling with color data [S2, unofficial].
- **Precision:** Normalization of 8-bit integers to floats requires care to avoid division by zero or incorrect scaling [S1, S2, unofficial].

## Subtopics

### Hierarchical Octree Voxel Coloring
Starts with a low-resolution voxel structure and refines it to higher resolutions [S3, unofficial]. This approach speeds up processing, taking approximately 15 seconds to generate a reconstruction with 3,256 voxels [S3, unofficial]. It differs from uniform resolution methods by adapting detail based on perceptual saliency [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on tree depth and traversal cost. Hierarchical variants reduce processing time significantly (e.g., 15s for 3,256 voxels) compared to full-resolution uniform methods which may take tens of minutes [S3, unofficial].
- **Memory Complexity:** Proportional to the number of non-empty leaf nodes in the SVO, storing packed color and normal data [S1, S2, unofficial].
- **Strengths:** Efficient retrieval of shaded colors directly from the SVO structure; supports hierarchical refinement for faster processing [S1, S2, S3, unofficial].
- **Limitations:** Requires careful handling of bit-packing and tree navigation; lighting is simple (directional) in the cited implementations [S1, S2, unofficial].

## Background
Originated from voxel coloring techniques for 3D reconstruction from uncalibrated camera views [S3, unofficial]. Prock and Dyer [ProDye98] utilized hierarchical octree representations to speed up voxel coloring [S3, unofficial]. The specific bit-packing and shading methods described in [S1, S2, unofficial] are found in modern implementations like `voxl` and `origin` [S1, S2, unofficial].

## Sources
[S1] rools/voxl, https://github.com/rools/voxl, fetched 2026-10-26 (unofficial)
[S2] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
[S3] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
