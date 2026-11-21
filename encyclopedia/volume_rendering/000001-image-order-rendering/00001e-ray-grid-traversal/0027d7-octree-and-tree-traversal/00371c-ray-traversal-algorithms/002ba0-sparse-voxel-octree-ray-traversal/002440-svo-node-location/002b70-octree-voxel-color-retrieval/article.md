# Octree Voxel Color Retrieval

Also called: Color Lookup with Shading, Voxel Lighting and Color Retrieval, lookupColor

## Overview
Octree Voxel Color Retrieval computes the final color of a hit voxel by retrieving its stored base color and normal from a Sparse Voxel Octree (SVO) and applying a directional lighting model [S1, S2, unofficial]. It differs from [Octree Node Lookup by Voxel](5120-octree-node-lookup-by-voxel.md) by explicitly performing shading calculations (ambient and diffuse) on the retrieved data rather than returning raw texture or color values [S1, S2, unofficial].

## How it works
**Inputs:**
*   Ray hit position, normal, and tree state from [SVO Node Location](3249-svo-node-location.md).
*   Octree data structure containing color, normal, and metadata.
*   Light direction vector $L$.
*   Shading mode flag (shadeless or shaded).

**Outputs:**
*   Final voxel color $C_{final}$ (RGB).

**Data Structures:**
*   **Octree Data:** A flat array of integers/floats.
*   **Voxel Color:** Stored as 8-bit integers per channel (R, G, B) packed into a 32-bit integer or separate components, normalized by a divisor $d$ [S2, unofficial].
*   **Voxel Normal:** Stored as 8-bit signed integers, normalized by dividing by 128.0 [S1, unofficial].
*   **Light Direction:** Defined as $L = \{-0.039030, -0.866792, 0.497139\}$ [S1, unofficial].

**Steps:**

1.  **Navigate to Color Data:**
    Using the node index and child index from the traversal state, navigate the octree hierarchy to find the leaf node containing the color data [S2, unofficial].
    *   Extract position and tree state: `px`, `node`, `cidx`, `level` [S2, unofficial].
    *   Calculate page header and block info to locate the palette node [S2, unofficial].
    *   Loop while the current node does not have color data (check bit `cidx` of `paletteNode`), incrementing level and updating pointers until the color-bearing leaf is found [S2, unofficial].
    *   Calculate the attach offset to retrieve the raw color component $c$ [S2, unofficial].

2.  **Extract and Normalize Base Color:**
    *   Extract RGB components from $c$:
        $$ r = (c \gg 16) \ \& \ 0xFF $$
        $$ g = (c \gg 8) \ \& \ 0xFF $$
        $$ b = c \ \& \ 0xFF $$
    *   Normalize to $[0, 1]$:
        $$ C_{base} = \left( \frac{r}{d}, \frac{g}{d}, \frac{b}{d} \right) $$
        Where $d$ is the normalization divisor [S2, unofficial].

3.  **Handle Shadeless Mode:**
    If the `shadeless` flag is set, return $C_{base}$ with alpha 1.0 and terminate [S2, unofficial].

4.  **Transform Normal to World Space:**
    *   Load the transformation matrix $M_{octree \to world}$ from the render list offsets [S2, unofficial].
    *   Transform the hit normal $N_{local}$ (from traversal) to world space:
        $$ N_{world} = \text{normalize}(M_{octree \to world} \times N_{local}) $$
        [S2, unofficial]

5.  **Compute Lighting:**
    *   **Ambient Component:**
        $$ C_{ambient} = S_{ambient} \times C_{light} $$
        Where $S_{ambient}$ is ambient strength and $C_{light}$ is light color [S2, unofficial].
    *   **Diffuse Component:**
        $$ C_{diffuse} = \max(N_{world} \cdot L, 0.0) \times C_{light} $$
        Where $L$ is the normalized light direction [S2, unofficial].
    *   **Final Color:**
        $$ C_{final} = (C_{ambient} + C_{diffuse}) \times C_{base} $$
        [S2, unofficial]

    *Alternative Normalization (Source [S1, unofficial]):*
    Some implementations compute brightness $B$ using the dot product of the normalized normal and light direction:
    $$ B = \frac{\arccos\left( \frac{N_x}{128.0} L_x + \frac{N_y}{128.0} L_y + \frac{N_z}{128.0} L_z \right)}{\pi} $$
    Then apply to color:
    $$ R = \text{uint}(V_{color}[1] \times B) $$
    $$ G = \text{uint}(V_{color}[1] \times B) $$
    $$ B_{val} = \text{uint}(V_{color}[0] \times B) $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octree_voxel_color_retrieval(hit_pos, hit_normal, tree_state, octree_data, light_dir, shadeless, ambient_strength, light_color):
    # 1. Navigate to color data
    node = tree_state.node
    cidx = tree_state.childIdx
    level = tree_state.scale
    
    # Calculate page header and block info
    page_header = node & 0xFFFFF800
    block_info = page_header + octree_data[page_header]
    attach_data = int(block_info) + octree_data.blockInfoEnd
    
    # Get palette node
    palette_node = octree_data[attach_data + int(node) - render_list.offsets[index] / 4 - 1]
    
    # Loop until color data is found
    while ((int(palette_node >> cidx) & 1) != 1):
        level += 1
        # Update bounds, node, cidx, pageHeader, blockInfo, attachData, paletteNode
        # [G] Implementation details for tree navigation loop omitted as per source structure
        pass 
        
    # Calculate attach offset and get color component
    attach_offset = attach_data + int(palette_node >> 8) + bit_count(palette_node & ((1 << cidx) - 1))
    c = int(octree_data[attach_offset])
    
    # 2. Extract and normalize base color
    r = (c >> 16) & 0xFF
    g = (c >> 8) & 0xFF
    b = c & 0xFF
    d = 255.0 # Typical divisor for 8-bit normalization
    octree_color = vec3(r / d, g / d, b / d)
    
    # 3. Handle shadeless mode
    if shadeless:
        return vec4(octree_color, 1.0)
        
    # 4. Transform normal to world space
    offset = int(render_list.offsets[index] + ubo.pageBytes) / 4 - ubo.transformCount * 4
    v = octree_data[offset : offset + 16] # Read 16 floats
    octree_to_world = mat4(v)
    hit_normal_world = normalize(octree_to_world * vec4(hit_normal, 0.0))
    
    # 5. Compute lighting
    ambient = ambient_strength * light_color
    light_dir_normalized = normalize(light_dir)
    diffuse = max(dot(hit_normal_world, light_dir_normalized), 0.0) * light_color
    
    final_color = (ambient + diffuse) * octree_color
    return final_color
```

## Subtopics

### Hierarchical Octree Voxel Coloring
Starts with a low-resolution voxel structure and refines it to higher resolutions [S3, unofficial]. Processing time is approximately 15 seconds for a reconstruction with 3,256 voxels [S3, unofficial]. Unlike uniform resolution methods, it may introduce perceptual saliency components [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the depth of the octree traversal to find the leaf node [S2, unofficial].
*   **Memory:** Depends on the SVO structure size; color data is stored per leaf [S2, unofficial].
*   **Performance:** Hierarchical approaches can reduce processing time for large reconstructions (e.g., 15s for 3,256 voxels) [S3, unofficial].
*   **Strengths:** Efficient retrieval of pre-shaded or dynamically shaded voxel colors; supports hierarchical refinement [S3, unofficial].
*   **Limitations:** Requires careful handling of coordinate transformations and normal storage formats (8-bit vs float) [S1, S2, unofficial].

## Background
Originated in the context of volumetric reconstruction from uncalibrated camera views, specifically utilizing hierarchical octree representations to speed up voxel coloring [S3, unofficial]. Prock and Dyer [ProDye98] utilized this approach to refine low-resolution structures to higher resolutions [S3, unofficial].

## Sources
[S1] rools/voxl, https://github.com/rools/voxl, fetched 2026-10-26 (unofficial)
[S2] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
[S3] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
