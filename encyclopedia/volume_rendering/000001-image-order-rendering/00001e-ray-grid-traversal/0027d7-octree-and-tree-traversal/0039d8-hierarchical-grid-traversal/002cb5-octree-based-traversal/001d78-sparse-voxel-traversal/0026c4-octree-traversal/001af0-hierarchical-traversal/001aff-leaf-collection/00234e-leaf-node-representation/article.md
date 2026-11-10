# Leaf Node Representation

Also called: Leaf struct

## Overview
The Leaf Node Representation defines the data structure and logic for terminal nodes in sparse voxel octree traversal [S1, unofficial]. It encapsulates volumetric sample data alongside validity and leaf flags, providing methods for validity checking and color manipulation during ray marching [S1, unofficial].

## How it works
**Inputs and Outputs:** Not given in the sources.

**Data Structures:**
The leaf node stores a validity bitmask (`valid`), a leaf flag bitmask (`leaf`), and a contiguous array of 4-byte RGBA values (`rgba`) [S1, unofficial]. Each bit in `valid` and `leaf` corresponds to a specific octant or voxel within the block [S1, unofficial].

**Steps:**
1. **Validity Check:** The `valid()` method returns true if `rgba[3] != 0x00` [S1, unofficial].
2. **Leaf Detection:** The traversal checks the `leaf` bitmask to determine if the current octant is a terminal node [S1, unofficial].
3. **Color Application:** The `set_color(pixel, lightness)` function multiplies the RGB channels of the pixel by the `lightness` factor while leaving the alpha channel unmodified [S1, unofficial].
4. **Equality Check:** Two leaf nodes are compared using a bitwise comparison of their entire 4-byte RGBA blocks rather than element-wise comparison [S1, unofficial].

**Formulas:**
* Validity: `is_valid = (valid_mask >> octant_index) & 1` [S1, unofficial].
* Color update: `pixel.rgb = voxel.rgba.rgb * lightness` [S1, unofficial].

**Parameters:**
* `lightness`: Scalar multiplier for RGB channels. Typical value: Not given in the sources.
* `octant_index`: Integer index of the voxel within the leaf block.

**Edge Cases and Pitfalls:**
* Zero alpha (`rgba[3] == 0x00`) marks a voxel as invalid [S1, unofficial].
* Bitmask indexing requires `octant_index` to be within the valid range of the bitmask width [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_leaf(voxel, octant_index, pixel, lightness):
    # Check if the octant is marked as a leaf node
    is_leaf = (voxel.leaf >> octant_index) & 1
    if not is_leaf:
        return False

    # Retrieve RGBA data for the octant
    rgba = voxel.rgba[octant_index]

    # Validity check: returns true if alpha channel is non-zero
    if rgba[3] == 0x00:
        return False

    # Apply lighting: multiply RGB by lightness, keep alpha unchanged
    pixel[0] = rgba[0] * lightness
    pixel[1] = rgba[1] * lightness
    pixel[2] = rgba[2] * lightness
    pixel[3] = rgba[3]
    return True

def check_equality(node_a, node_b):
    # Bitwise comparison of the entire 4-byte RGBA block
    return node_a.rgba == node_b.rgba
```

## Subtopics

### Bitfield Histogram Culling
This variant stores a histogram of values contained in the spatial region represented by an octree node as a bitfield instead of transfer-function dependent visibility information [S2, unofficial]. The visible value range of a transfer function is also represented as a histogram [S2, unofficial]. To determine if a node is empty, its bitfield is tested against the transfer function's bitfield using a bitwise AND operation [S2, unofficial]. This allows transfer-function changes at run-time without reconstructing the acceleration structure [S2, unofficial]. The approach achieved best results with a leaf-node size of $4^3$ and optimized traversal by starting at a subdivision level other than the root node [S2, unofficial].

## Operating characteristics
* **Memory:** Leaf-node size of $4^3$ achieved best results [S2, unofficial].
* **Performance:** Bitwise operations are used for validity checks and culling [S1, S2, unofficial].
* **Limitations:** Not given in the sources.

## Background
The leaf node representation is derived from the Zazen renderer [S1, unofficial]. The bitfield histogram culling variant was proposed by Faludi et al. [S2, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
