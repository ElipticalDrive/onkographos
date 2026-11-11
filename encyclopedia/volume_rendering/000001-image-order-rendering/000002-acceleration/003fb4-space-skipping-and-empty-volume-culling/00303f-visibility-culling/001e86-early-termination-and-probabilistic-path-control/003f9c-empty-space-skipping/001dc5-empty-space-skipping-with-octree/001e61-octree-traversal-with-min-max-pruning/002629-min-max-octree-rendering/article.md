# Min-Max Octree Rendering

Also called: Unclassified Volume Rendering, Unclassified Volume Rendering with Min-Max Octree, vpRenderClassifiedVolume, vpRenderRawVolume, vpRenderRawVolume algorithm

## Overview
Min-Max Octree Rendering is a volume rendering acceleration technique that uses a hierarchical data structure storing the minimum and maximum scalar values of voxel blocks to accelerate rendering unclassified volumes [S1]. It solves the performance bottleneck of frequent classification adjustments by allowing the underlying scalar volume to remain "raw" (unclassified) while the octree enables fast culling during ray traversal [S1, S2]. It is distinct from [Classified Volume Rendering](1738-classified-volume-rendering.md) because it avoids the $O(N)$ cost of precomputing opacity for every voxel, making it suitable when transfer functions change frequently, though it is slower than classified rendering due to per-voxel classification during traversal [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar voxel array (raw volume), classification tables (transfer functions), view parameters, and shading parameters [S1, S2].
*   **Output:** A 2D image array (framebuffer) [S2].

**Data Structures:**
*   **Min-Max Octree:** A hierarchical structure where each node stores the minimum and maximum scalar values found within its corresponding voxel block [S1]. The tree is built recursively. The smallest node size is defined by `base_node_size` [S1].
*   **Unclassified Volume:** The raw scalar data, defined by size, voxel spacing, and field indices [S1].

**Algorithm Steps:**
1.  **Define Volume:** Initialize the unclassified volume context using `vpSetVolumeSize()`, `vpSetVoxelSize()`, `vpSetVoxelField()`, and `vpSetRawVoxels()` [S1]. Ensure the voxel data array is populated [S1].
2.  **Configure Octree Resolution:** Set the `base_node_size` parameter, which defines the side length (in voxels) of the smallest octree nodes. A typical value is 4; smaller values increase resolution at the expense of increased size [S1].
3.  **Compute Min-Max Octree:** Call `vpCreateMinMaxOctree(vpc, root_node_size, base_node_size)` [S1]. This routine reads the raw voxel data, computes the min/max bounds for each node, and stores the octree in the rendering context [S1].
    *   *Note:* `root_node_size` is reserved and currently unused [S1].
4.  **Set Rendering Parameters:**
    *   Set view and shading parameters [S1].
    *   Set the maximum opacity threshold (`VP_MAX_RAY_OPACITY`), the opacity value at which ray compositing stops. A typical value is 0.95 [S2].
    *   (Optional) Set intermediate image size hints (`VP_INT_WIDTH_HINT`, `VP_INT_HEIGHT_HINT`) to pre-allocate work buffers and avoid reallocation overhead [S2].
5.  **Render:** Call `vpRenderRawVolume(vpc)` [S1].
    *   The renderer traverses the ray through the min-max octree [S2].
    *   Voxels within active nodes are classified on-the-fly using the current transfer function and composited [S1].
6.  **Update:** If the raw voxel data changes, step 3 must be repeated to recompute the octree, otherwise renderings will be incorrect. If the transfer function changes, only step 5 is needed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class MinMaxNode:
    def __init__(self, min_val, max_val, size, children=None):
        self.min = min_val
        self.max = max_val
        self.size = size
        self.children = children if children else []

def build_min_max_octree(data, base_node_size):
    """Recursively builds a min-max octree from a 3D scalar array."""
    def recurse(arr, current_size):
        if current_size <= base_node_size:
            return MinMaxNode(arr.min(), arr.max(), current_size)
        mid = current_size // 2
        children = []
        for i in range(2):
            for j in range(2):
                for k in range(2):
                    slice_arr = arr[i*mid:(i+1)*mid, j*mid:(j+1)*mid, k*mid:(k+1)*mid]
                    children.append(recurse(slice_arr, mid))
        return MinMaxNode(
            min(c.min for c in children),
            max(c.max for c in children),
            current_size,
            children
        )
    return recurse(data, data.shape[0])

class Context:
    def __init__(self):
        self.volume_size = (0, 0, 0)
        self.voxel_size = (1.0, 1.0, 1.0)
        self.raw_voxels = None
        self.octree = None
        self.view_params = None
        self.max_opacity = 1.0
        self.transfer_function = None

class TransferFunction:
    def __init__(self, min_active, max_active):
        self.min_active = min_active
        self.max_active = max_active
    def classify(self, val):
        # Standard transfer function lookup and compositing weights [G]
        return np.array([0.0, 0.0, 0.0]), 0.0

def vpRenderRawVolume(context):
    """Complete rendering loop using min-max octree culling and front-to-back compositing."""
    width, height, depth = context.volume_size
    dx, dy, dz = context.voxel_size
    origin, direction = context.view_params.ray_setup()
    image = np.zeros((height, width, 3))
    max_opacity = context.max_opacity
    transfer = context.transfer_function

    for y in range(height):
        for x in range(width):
            ray_origin = origin + x * direction * dx + y * direction * dy
            ray_dir = direction
            accum_color = np.array([0.0, 0.0, 0.0])
            accum_opacity = 0.0
            t = 0.0
            step = 1.0 / depth

            while t < 1.0:
                voxel_pos = ray_origin + t * ray_dir
                if not (0 <= voxel_pos.x < width and 0 <= voxel_pos.y < height and 0 <= voxel_pos.z < depth):
                    break

                node = context.octree.traverse(voxel_pos)
                if node.min > transfer.max_active or node.max < transfer.min_active:
                    t += node.size * step
                    continue

                idx = (int(voxel_pos.x), int(voxel_pos.y), int(voxel_pos.z))
                voxel_val = context.raw_voxels[idx]
                color, opacity = transfer.classify(voxel_val)

                alpha = opacity * (1.0 - accum_opacity)
                accum_color += color * alpha
                accum_opacity += alpha

                if accum_opacity >= max_opacity:
                    break

                t += step

            image[y, x] = accum_color
    return image
```

**Formulas and Parameters:**
*   **Intermediate Image Size Bound:** If the viewing transformation includes no scaling, the maximum intermediate image dimension is $2 \times \max(\text{volume dimension})$ [S2].
*   **Base Node Size:** Controls the granularity of the octree. Smaller `base_node_size` $\rightarrow$ better pruning resolution $\rightarrow$ higher memory cost [S1].
*   **Range Parameter (Historical/Contextual):** In older VolPack versions, `vpMinMaxOctreeThreshold` accepted a `range` parameter defining the table index variation considered "small" for pruning [S1].

**Edge Cases and Pitfalls:**
*   **Data Staleness:** If the raw voxel array is modified, `vpCreateMinMaxOctree()` must be called again. Failure to do so results in incorrect renderings using stale bounds [S1].
*   **Performance Trade-off:** While faster than no octree, this method is slower than [Classified Volume Rendering](1738-classified-volume-rendering.md) because classification occurs per-voxel during traversal rather than being precomputed [S2].
*   **Error Conditions:**
    *   `VPERROR_BAD_SHADER`: Invalid shading table sizes or voxel fields [S2].
    *   `VPERROR_SINGULAR`: Singular view transformation matrix [S2].
    *   `VPERROR_BAD_VOLUME`: Volume size not set or invalid [S2].
    *   `VPERROR_BAD_CLASSIFIER`: Invalid opacity transfer function tables (specific to `vpRenderRawVolume`) [S2].
    *   `VPERROR_BAD_SHADOW`: Angle between shadow light vector and view direction $\ge 45^\circ$ [S2].

## Subtopics

### Rain-Max Octree Classification
A data structure for encoding spatial coherence in unclassified volumes (scalar fields with no precomputed opacity) [S3]. It allows classification and rendering without an expensive preprocessing step when the opacity transfer function changes [S3]. Unlike previous algorithms that employ spatial data structures and require an expensive preprocessing step when the opacity transfer function changes, the rain-max octree avoids this step by being classification-independent [S3]. When combined with the shear-warp rendering algorithm, it can classify and render a $256^3$ voxel volume in three seconds [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Rendering with an octree is significantly faster than without, but still slower than rendering with `vpRenderClassifiedVolume` [S2].
*   **Strengths:** Should be used when the volume data or the opacity transfer function changes in between renderings [S2].
*   **Limitations:** Slower than classified rendering for static transfer functions.
*   **When to Use:** When volume data is static but classification/transfer functions change often, or when the volume is sparse enough that min-max pruning provides significant culling [S1, S2].
*   **When Not to Use:** When the transfer function is static (use Classified Volume Rendering) or when a single rendering is needed with no precomputation allowed (use raw rendering without octree) [S1].

## Background
Min-Max Octree Rendering was introduced as part of the VolPack library [S1, S2]. It was designed to address the limitations of pre-classified rendering in interactive visualization scenarios where users need to adjust opacity transfer functions in real-time [S1, S2]. The technique leverages the spatial coherence of volume data by summarizing voxel values in octree nodes, allowing for efficient empty-space skipping and isovalue range culling [S1]. It is implemented in the `vpRenderRawVolume` routine of VolPack [S1, S2].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] man vpRenderRawVolume (3): render a volume, https://manpages.org/vprenderrawvolume/3, fetched 2026-10-30
[S3] PDF Fast Volume Rendering Using a Shear-warp Factorization of The Viewing ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
