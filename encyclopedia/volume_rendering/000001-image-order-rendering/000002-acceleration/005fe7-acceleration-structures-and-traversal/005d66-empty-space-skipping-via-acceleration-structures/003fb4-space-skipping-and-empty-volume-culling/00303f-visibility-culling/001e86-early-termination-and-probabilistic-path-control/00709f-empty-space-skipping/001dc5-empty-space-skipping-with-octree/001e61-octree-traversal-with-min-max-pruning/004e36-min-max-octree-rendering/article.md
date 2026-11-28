# Min-Max Octree Rendering

Also called: Min-max octree entry points

## Overview
Min-Max Octree Rendering is an acceleration technique for volume rendering that uses a hierarchical data structure storing the minimum and maximum scalar values of voxel regions to prune transparent space and determine precise ray entry points [S1][S3, unofficial]. It is specifically designed for unclassified volumes where the opacity transfer function or classification parameters change frequently, avoiding the expensive preprocessing required by pre-classified rendering methods [S1, S4]. Unlike block-based entry point determination, it operates at a finer resolution than block granularity, significantly reducing the number of rays intersecting transparent regions [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Unclassified volume data (scalar field), rendering context (view, shading, classification tables), and a precomputed min-max octree.
*   **Output:** Rendered image array.

**Data Structures:**
1.  **Min-Max Octree:** A hierarchical structure where each node stores the minimum and maximum values of the voxel field for its region [S1][S3, unofficial].
    *   **Layout:** Linearized octree bit encoding using hierarchy compression [S3, unofficial].
    *   **Node Content:** 2-byte minimum and 2-byte maximum values (4 bytes total per node) [S3, unofficial].
    *   **Indexing:** Nodes are indexed by `index(node) = \sum_{i=0}^{N-1} octanti(node) \cdot 8^{N - i - 1}`, where $N$ is the octree depth and $octanti(node)$ is the octant of level $i$ [S3, unofficial]. There are $8^N$ possible indices for depth $N$ [S3, unofficial].
2.  **Entry Point Buffer:** Stores the block index and the corresponding octree node index for each ray [S3, unofficial].
3.  **Lookup Table:** Precomputed relative translational offsets for octree nodes, sized $8^N$ entries [S3, unofficial].

**Parameters:**
*   **base_node_size:** Size in voxels of one side of the smallest node in the octree. Controls resolution vs. memory trade-off. Typical value: 4 [S1].
*   **range:** Range of table indices considered "small" for classification sensitivity. Affects pruning effectiveness [S1].
*   **max_opacity_threshold:** Threshold for pixel opacity to stop compositing. Typical value: 0.95 [S2].
*   **intermediate_image_size:** Hints for work buffer allocation (`VP_INT_WIDTH_HINT`, `VP_INT_HEIGHT_HINT`) [S2].

**Algorithm Steps:**
1.  **Initialization:** Define the unclassified volume using `vpSetVolumeSize()`, `vpSetVoxelSize()`, `vpSetVoxelField()`, and `vpSetRawVoxels()` [S1]. Initialize volume data [S1].
2.  **Octree Construction:**
    *   For each classification table, call `vpMinMaxOctreeThreshold(param_num, range)` to define sensitivity [S1].
    *   Call `vpCreateMinMaxOctree(vpc, root_node_size, base_node_size)` to compute the octree from the raw volume data [S1].
    *   *Note:* If volume data changes, `vpCreateMinMaxOctree()` must be called again [S1].
3.  **Rendering Loop:**
    *   Set view and shading parameters [S1].
    *   Choose classification function (opacity transfer function) [S1].
    *   Call `vpRenderRawVolume()` [S1].

**Detailed Rendering Logic (from [S3, unofficial]):**
1.  **Entry Point Determination:**
    *   Use the min-max octree to find the first non-transparent node along the ray.
    *   The entry point buffer stores the block index and octree node index [S3, unofficial].
    *   Calculate the ray entry depth: $d_{entry} = t_z + \text{offset}_{node}$, where $t_z$ is the translation vector component and $\text{offset}_{node}$ is retrieved from the lookup table using `index(node)` [S3, unofficial].
2.  **Projection:**
    *   Create a template for every octree level [S3, unofficial].
    *   Recursively traverse the hierarchical classification information in back-to-front order [S3, unofficial].
    *   For each node, if it is non-transparent (based on min/max values vs. classification thresholds), project the appropriate template for that level [S3, unofficial].
3.  **Compositing:**
    *   Compositing proceeds from the determined entry point.
    *   Stop compositing when the accumulated opacity reaches `max_opacity_threshold` [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_min_max_octree(volume_data, context):
    # 1. Initialization: Define unclassified volume
    set_volume_size(context, volume_data.size)
    set_voxel_size(context, volume_data.voxel_size)
    set_voxel_field(context, volume_data.field)
    set_raw_voxels(context, volume_data.array)
    
    # 2. Octree Construction
    # Define classification sensitivity for a parameter
    vp_min_max_octree_threshold(context, param_num=0, range=10)
    
    # Build the octree (root_node_size is reserved, typically 0)
    vp_create_min_max_octree(context, root_node_size=0, base_node_size=4)
    
    # 3. Rendering Loop
    while rendering_needed(context):
        set_view_and_shading(context)
        set_opacity_transfer_function(context, context.current_transfer_function)
        
        # Render using the octree for acceleration
        vp_render_raw_volume(context)

def vp_render_raw_volume(context):
    # Get intermediate image buffer size hints if not set
    if not has_intermediate_hints(context):
        max_dim = 2 * max(context.volume_dimensions)
        set_state(context, "VP_INT_WIDTH_HINT", max_dim)
        set_state(context, "VP_INT_HEIGHT_HINT", max_dim)
        
    # Initialize entry point buffer
    entry_points = allocate_entry_point_buffer()
    
    # For each ray in the view:
    for ray in context.rays:
        # Traverse min-max octree to find first non-transparent node
        node = traverse_octree_for_entry(ray, context.octree)
        
        if node is not None:
            # Calculate entry depth: tz + relative offset
            node_idx = calculate_node_index(node)
            offset = lookup_relative_offset(node_idx)
            entry_depth = ray.tz + offset
            
            # Store in buffer
            entry_points[ray.id] = (node.block_index, node_idx, entry_depth)
            
            # Compositing from entry_depth
            opacity = 0.0
            final_color = 0.0
            for sample in ray.samples_from(entry_depth):
                value = sample.value
                alpha = context.transfer_function(value)
                opacity += alpha * (1.0 - opacity)
                final_color += alpha * (1.0 - opacity) * sample.color
                if opacity >= context.max_opacity_threshold:
                    break
            entry_points[ray.id].final_color = final_color

# Helper definitions
def set_volume_size(context, size): context.volume_size = size
def set_voxel_size(context, size): context.voxel_size = size
def set_voxel_field(context, field): context.voxel_field = field
def set_raw_voxels(context, data): context.raw_voxels = data
def vp_min_max_octree_threshold(context, param_num, range): context.classification_range[param_num] = range
def vp_create_min_max_octree(context, root_node_size, base_node_size): context.octree = compute_min_max_octree(context.raw_voxels, base_node_size)
def rendering_needed(context): return True
def set_view_and_shading(context): pass
def set_opacity_transfer_function(context, func): context.transfer_function = func
def has_intermediate_hints(context): return "intermediate_hints" in context
def set_state(context, key, value): context.state[key] = value
def allocate_entry_point_buffer(): return {}
def traverse_octree_for_entry(ray, octree): return octree.find_first_non_transparent(ray)
def calculate_node_index(node):
    idx = 0
    for i, octant in enumerate(node.octants):
        idx += octant * (8 ** (node.depth - i - 1))
    return idx
def lookup_relative_offset(node_idx): return context.offset_table[node_idx]
```

**Pitfalls and Errors:**
*   Shading lookup tables sized incorrectly, invalid voxel fields, or invalid material types cause `VPERROR_BAD_SHADER` [S2].
*   Singular view transformation matrix causes `VPERROR_SINGULAR` [S2].
*   Volume size not set, invalid, or missing data causes `VPERROR_BAD_VOLUME` [S2].
*   Opacity transfer function tables have invalid sizes or are incorrectly specified (render raw only) causing `VPERROR_BAD_CLASSIFIER` [S2].
*   Angle between shadow light vector and viewing direction too large (>45°) causes `VPERROR_BAD_SHADOW`; fix by changing light direction or disabling shadows [S2].

## Subtopics

### Rain-Max Octree Classification
A classification-independent data structure for encoding spatial coherence in unclassified volumes [S4]. Unlike standard min-max octrees which may require preprocessing when opacity changes, the rain-max octree allows classification and rendering without an expensive preprocessing step [S4]. It enables classifying and rendering a $256^3$ voxel volume in three seconds when combined with shear-warp rendering [S4].

## Operating characteristics
*   **Time Complexity:** Rendering is significantly faster than rendering without an octree, but slower than rendering with pre-classified volumes (`vpRenderClassifiedVolume`) [S2].
*   **Memory Complexity:** The min-max octree for depth 3 with 4 bytes per node requires at most 2340 bytes; classification information requires 66 bytes [S3, unofficial]. Total memory overhead is approximately 10% for typical datasets (e.g., $32^3$ blocks) [S3, unofficial].
*   **Strengths:** Ideal for interactive exploration where opacity transfer functions or classification parameters change frequently [S1, S2]. Provides finer-grained culling than block-based methods [S3, unofficial].
*   **Limitations:** Requires preprocessing time to build the octree if the volume data changes [S1]. Slower than pre-classified rendering for static classification [S2].
*   **When to use:** When volume data or opacity transfer function changes between renderings [S2].
*   **When not to use:** When a single rendering is needed (raw volume rendering is faster due to no octree overhead) or when classification is static (pre-classified rendering is faster) [S1].

## Background
The min-max octree was introduced as part of the VolPack library to accelerate volume rendering of unclassified data [S1]. It extends block-based entry point determination by replacing block-level min/max values with an octree structure [S3, unofficial]. The concept was further refined in research on efficient visualization of large medical datasets, where it was combined with shear-warp factorization and cell invisibility caches [S3, unofficial][S4]. The "rain-max" variant was proposed to eliminate preprocessing costs associated with opacity changes [S4].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] man vpRenderRawVolume (3): render a volume, https://manpages.org/vprenderrawvolume/3, fetched 2026-10-30
[S3] Efficient Volume Visualization of Large Medical Datasets, https://old.cescg.org/CESCG-2004/web/Bruckner-Stefan/html/, fetched 2026-10-30 (unofficial)
[S4] PDF Fast Volume Rendering Using a Shear-warp Factorization of The Viewing ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
