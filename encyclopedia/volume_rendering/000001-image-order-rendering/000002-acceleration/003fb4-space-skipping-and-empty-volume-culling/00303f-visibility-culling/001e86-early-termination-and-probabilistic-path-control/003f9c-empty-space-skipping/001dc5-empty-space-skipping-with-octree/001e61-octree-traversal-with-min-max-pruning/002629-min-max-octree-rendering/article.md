# Min-Max Octree Rendering

Also called: Unclassified Volume Rendering, Unclassified Volume Rendering with Min-Max Octree, vpRenderClassifiedVolume, vpRenderRawVolume, vpRenderRawVolume algorithm

## Overview
Min-Max Octree Rendering is a volume rendering acceleration technique designed for unclassified scalar volumes where the opacity transfer function or volume data changes frequently between renderings [S1, S2]. It solves the performance bottleneck of recomputing classified volumes by precomputing a hierarchical min-max octree that stores the minimum and maximum scalar values for each node [S1]. This allows the renderer to skip traversal of nodes that do not contain the target isovalue range, significantly accelerating rendering compared to raw voxel traversal, though it remains slower than pre-classified rendering [S2]. It is distinct from [Octree Traversal with Min/Max Pruning](1739-octree-traversal-with-min-max-pruning.md) in that it is specifically integrated into the VolPack API for unclassified data management and opacity transfer function flexibility [S1, S2].

## How it works
The method relies on the VolPack library context to manage volume data, classification tables, and the min-max octree structure [S1, S2].

**Inputs and Outputs:**
*   **Input:** Unclassified scalar volume data (voxel array), opacity transfer function (classification table), view parameters, shading parameters, and a precomputed min-max octree [S1, S2].
*   **Output:** Rendered image stored in the current image array [S2].

**Data Structures:**
*   **Unclassified Volume:** Defined by size, voxel size, voxel field, and raw voxel data [S1].
*   **Min-Max Octree:** A hierarchical data structure where each node stores the minimum and maximum scalar values of the voxels contained within its bounding box [S1].
    *   **Node Size:** The smallest nodes in the octree have a side length of `base_node_size` voxels [S1].
*   **Intermediate Image:** A work buffer used to hold temporary results during rendering [S2].

**Parameters:**
*   `base_node_size`: Size in voxels of one side of the smallest node in the octree. Smaller values provide better resolution (more pruning) at the expense of increased memory size. Typical value: 4 [S1].
*   `range`: A range of table indices for a parameter considered 'small'. Used during octree construction to determine if a node's value range is narrow enough to be pruned. Choosing a value too small or too large may reduce performance benefit [S1].
*   `VP_MAX_RAY_OPACITY`: Maximum opacity threshold for a pixel. If reached, no more voxels are composited. Typical value: 0.95. Default: 1.0 [S2].
*   `VP_INT_WIDTH_HINT` / `VP_INT_HEIGHT_HINT`: Hints for the maximum size of the intermediate image work buffer to avoid reallocation overhead [S2].

**Steps:**
1.  **Initialize Context:** Create a VolPack context [S2].
2.  **Define Volume:** Define the unclassified volume using `vpSetVolumeSize()`, `vpSetVoxelSize()`, `vpSetVoxelField()`, and `vpSetRawVoxels()`. Initialize the volume data [S1].
3.  **Precompute Octree:**
    *   For each classification table, call `vpMinMaxOctreeThreshold(vpc, param_num, range)` to specify the range of indices considered 'small' for that parameter [S1].
    *   Call `vpCreateMinMaxOctree(vpc, root_node_size, base_node_size)` to compute the octree from the unclassified volume data. `root_node_size` is reserved; `base_node_size` controls granularity [S1].
4.  **Set Rendering Parameters:**
    *   Choose the classification function (opacity transfer function) [S1].
    *   Set view and shading parameters [S1].
    *   Set `VP_MAX_RAY_OPACITY` if a threshold other than 1.0 is desired [S2].
    *   (Optional) Set `VP_INT_WIDTH_HINT` and `VP_INT_HEIGHT_HINT` to allocate the intermediate image buffer once [S2].
5.  **Render:** Call `vpRenderRawVolume(vpc)` [S1, S2].
    *   The function uses the min-max octree automatically if present to accelerate rendering [S2].
    *   If the volume data changes, step 3 must be repeated to recompute the octree [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize VolPack context
vpc = vpCreateContext()

# Define the unclassified volume
vpSetVolumeSize(vpc, width, height, depth)
vpSetVoxelSize(vpc, voxel_width, voxel_height, voxel_depth)
vpSetVoxelField(vpc, field_index)
vpSetRawVoxels(vpc, voxel_array) # voxel_array contains scalar values

# Precompute the min-max octree
# Specify the 'small' range for the classification parameter
# range is the number of table indices considered similar in opacity
vpMinMaxOctreeThreshold(vpc, param_num=0, range=5)

# Create the octree with base node size 4
vpCreateMinMaxOctree(vpc, root_node_size=0, base_node_size=4)

# Set rendering parameters
vpSetClassifierTable(vpc, param_num=0, table_data) # Opacity transfer function
vpSetViewTransform(vpc, view_matrix)
vpSetShadingParameters(vpc, shading_params)

# Set max opacity threshold (optional, default 1.0)
vpSetd(vpc, VP_MAX_RAY_OPACITY, 0.95)

# Set intermediate image size hints (optional)
# Max dimension bound without scaling is 2 * max(volume dimension)
vpSeti(vpc, VP_INT_WIDTH_HINT, 2 * max(width, height, depth))
vpSeti(vpc, VP_INT_HEIGHT_HINT, 2 * max(width, height, depth))

# Render the volume
# This uses the min-max octree for acceleration
result = vpRenderRawVolume(vpc)

# If volume data changes, recompute octree
# vpCreateMinMaxOctree(vpc, 0, 4)

# Cleanup
vpDestroyMinMaxOctree(vpc)
vpDestroyContext(vpc)
```

**Edge Cases and Pitfalls:**
*   **Data Changes:** If the volume array contents change, `vpCreateMinMaxOctree()` must be called again, or renderings will be incorrect [S1].
*   **Range Selection:** The `range` parameter in `vpMinMaxOctreeThreshold` affects performance. Too small or too large values reduce the benefit [S1].
*   **Intermediate Image Size:** If the viewing transformation includes scaling, the intermediate image may need to be larger than the volume dimensions. The maximum size without scaling is $2 \times \max(\text{volume dimension})$ [S2].
*   **Errors:**
    *   `VPERROR_BAD_SHADER`: Incorrect shading lookup table sizes, invalid voxel fields, or invalid material types [S2].
    *   `VPERROR_SINGULAR`: Singular view transformation matrix [S2].
    *   `VPERROR_BAD_VOLUME`: Volume size not set, invalid, or no volume data [S2].
    *   `VPERROR_BAD_CLASSIFIER`: Invalid opacity transfer function tables or associations (vpRenderRawVolume only) [S2].
    *   `VPERROR_BAD_SHADOW`: Angle between shadow light vector and viewing direction $\ge 45^\circ$ [S2].

## Subtopics

## Operating characteristics
*   **Performance:** Rendering with a min-max octree is significantly faster than rendering without an octree (raw traversal) [S2]. However, it is still slower than rendering with `vpRenderClassifiedVolume` [S2].
*   **Use Case:** Best suited for scenarios where the volume data or the opacity transfer function changes frequently between renderings [S1, S2]. It avoids the expensive preprocessing step of recomputing a classified volume for every change [S1].
*   **Memory:** The min-max octree increases memory usage. Smaller `base_node_size` values improve pruning resolution but increase the size of the octree data structure [S1].
*   **Scaling:** Performance benefit depends on the `range` parameter and the nature of the volume data. The octree improves performance even with non-optimal range values, but experimentation is recommended [S1].

## Background
Min-Max Octree Rendering is part of the VolPack library, developed by the Stanford Visualization Group [S1]. It was introduced as an alternative to classified volume rendering for applications requiring frequent adjustments to opacity transfer functions or volume data [S1]. The method leverages the hierarchical min-max structure to prune empty or transparent regions during ray traversal, a concept related to [Octree Traversal with Min/Max Pruning](1739-octree-traversal-with-min-max-pruning.md) but implemented within the VolPack API's specific context management system [S1, S2].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] man vpRenderRawVolume (3): render a volume, https://manpages.org/vprenderrawvolume/3, fetched 2026-10-30
