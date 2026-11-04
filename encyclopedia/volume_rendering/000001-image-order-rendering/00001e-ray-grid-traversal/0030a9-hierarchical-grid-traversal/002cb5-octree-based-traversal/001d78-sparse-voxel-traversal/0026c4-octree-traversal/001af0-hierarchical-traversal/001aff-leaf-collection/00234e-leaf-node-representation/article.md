# Leaf Node Representation

Also called: Leaf struct

## Overview
The Leaf Node Representation defines the data structure and logic for the terminal nodes in a sparse voxel octree traversal. It encapsulates the volumetric data for a leaf block and provides methods for validity checking and color manipulation during ray marching. It differs from internal nodes by storing actual sample values rather than hierarchical pointers or aggregate statistics, serving as the primary source of radiance contribution in image-order rendering.

## How it works
**Inputs:** A pointer or reference to the leaf node data; a ray segment or voxel coordinate; optional lighting parameters.
**Outputs:** Updated pixel color (via accumulation), traversal state (continue/stop).

**Data Structure:**
The leaf node is typically a fixed-size block of voxels. In the reference implementation [S1, unofficial], the leaf node contains a validity bitmask and a leaf flag bitmask, alongside the voxel data. The voxel data itself is often stored as a contiguous array of RGBA values.
*   `valid`: A bitmask where each bit corresponds to a voxel in the leaf block. If bit $i$ is set, voxel $i$ is valid.
*   `leaf`: A bitmask indicating which octants are leaf nodes (in hierarchical structures).
*   `rgba`: An array of 4-byte unsigned integers (or floats) representing the color and opacity of each voxel.

**Steps:**
1.  **Validity Check:** Determine if the current voxel or leaf block is active.
    *   Formula: `is_valid = (valid_mask >> octant_index) & 1` [S1, unofficial].
    *   Specific check for empty space: `valid()` returns true if `rgba[3] != 0x00` [S1, unofficial]. This implies that a leaf node is considered "empty" or invalid for rendering if its alpha channel is zero.
2.  **Traversal Decision:** If the node is a leaf (indicated by the `leaf` bitmask), proceed to shading/compositing. If not, traverse children.
3.  **Color Application:** When shading, the base color is modified by lighting.
    *   Operation: `set_color(pixel, lightness)` multiplies the RGB channels of the pixel by the `lightness` factor. The alpha channel is not modified [S1, unofficial].
4.  **Equality Check:** For debugging or structural integrity, equality is determined by bitwise comparison of the entire 4-byte RGBA block, not element-wise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_leaf_node(leaf_node, ray, pixel, lightness):
    """
    Processes a leaf node during ray traversal.
    leaf_node: Object containing valid_mask, leaf_mask, and rgba array.
    ray: Current ray state.
    pixel: Current accumulated color buffer.
    lightness: Scalar lighting factor.
    """
    # Determine current octant/voxel index based on ray position
    octant_index = get_octant_index(ray, leaf_node)
    
    # Check validity using bitmask [S1, unofficial]
    is_valid = (leaf_node.valid_mask >> octant_index) & 1
    
    if not is_valid:
        return # Skip empty or invalid voxel

    # Retrieve RGBA data
    # Note: Exact indexing depends on memory layout (e.g., row-major)
    rgba_data = leaf_node.rgba[octant_index]
    
    # Check if the leaf node itself is considered "valid" by alpha [S1, unofficial]
    if rgba_data[3] == 0x00:
        return

    # Apply lighting [S1, unofficial]
    # set_color multiplies RGB by lightness, leaves alpha unchanged
    r = rgba_data[0] * lightness
    g = rgba_data[1] * lightness
    b = rgba_data[2] * lightness
    a = rgba_data[3]
    
    # Composite into pixel (standard front-to-back alpha blending)
    # pixel = pixel + (1 - pixel.a) * (r, g, b, a)
    # Implementation detail: [G] Standard alpha compositing assumed
    pixel.r = pixel.r + (1.0 - pixel.a) * r
    pixel.g = pixel.g + (1.0 - pixel.a) * g
    pixel.b = pixel.b + (1.0 - pixel.a) * b
    pixel.a = pixel.a + (1.0 - pixel.a) * a

def check_leaf_equality(leaf1, leaf2):
    """
    Checks equality by bitwise comparison of the entire RGBA block [S1, unofficial].
    """
    # Assuming rgba is stored as a contiguous 4-byte integer or struct
    return leaf1.rgba == leaf2.rgba
```

**Parameters:**
*   `lightness`: Scalar multiplier for RGB channels. Typical value: 1.0 (no change) or derived from lighting model.
*   `octant_index`: Integer index of the voxel within the leaf block.

**Edge Cases:**
*   **Zero Alpha:** A leaf node with `rgba[3] == 0x00` is treated as invalid/empty [S1, unofficial].
*   **Bitmask Bounds:** Ensure `octant_index` is within the bounds of the `valid_mask` size (e.g., 64 bits for 8x8x8 blocks).

## Subtopics

### Bitfield Histogram Culling
This variant stores a histogram of values in the spatial region as a bitfield instead of transfer-function dependent visibility information [S2, unofficial]. The visible value range of a transfer function is also represented as a histogram. To determine if a node is empty, its bitfield is tested against the transfer function's bitfield using a bitwise AND operation [S2, unofficial]. This allows for transfer-function changes at run-time without reconstructing the acceleration structure [S2, unofficial]. It achieved best results with a leaf-node size of $4^3$ and optimized traversal by starting at a subdivision level other than the root [S2, unofficial].

## Operating characteristics
*   **Memory:** Depends on leaf block size. A $4^3$ leaf with 4-byte RGBA values uses $64 \times 4 = 256$ bytes per leaf [S2, unofficial].
*   **Performance:** Bitwise operations for validity and culling are extremely fast [S1, S2, unofficial]. Histogram-based culling [S2, unofficial] avoids transfer-function reconstruction overhead.
*   **Limitations:** The validity check `rgba[3] != 0x00` [S1, unofficial] may be too coarse for nodes with partially transparent data. Bitwise equality [S1, unofficial] is efficient but requires exact memory layout matching.

## Background
The leaf node representation is a fundamental component of sparse voxel octrees used in volume rendering. The specific implementation details regarding validity checks and color manipulation are derived from the Zazen renderer [S1, unofficial]. The concept of using bitfield histograms for transfer-function independent culling was proposed by Faludi et al. [S2, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
