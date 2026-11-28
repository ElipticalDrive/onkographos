# Splatting-Based AMR Volume Rendering

Also called: Hierarchical Splatting for AMR, Hierarchical Splatting with k-d-tree and Octree, Park et al. method, Park, Bajaj & Siddavanahalli (2002) method, k-d-tree/octree hierarchical splatting

## Overview
Splatting-Based AMR Volume Rendering is a hierarchical object-order rendering technique designed for Adaptive Mesh Refinement (AMR) data [S1, S2, unofficial]. It solves the problem of efficiently rendering non-uniform, multi-resolution scalar fields by converting the irregular AMR hierarchy into a structured k-d-tree of constant-resolution blocks, augmented with octrees to manage isovalue ranges [S1, S2, unofficial]. Unlike standard splatting which treats primitives uniformly, this method uses the hierarchical structure to cull regions outside a specified isovalue range and transfer function, rendering only relevant data back-to-front [S1, S2, unofficial]. It sets itself apart from sibling methods like [Octree-based Splatting](8067-octree-based-splatting.md) by explicitly handling the conversion of AMR block structures into a k-d-tree framework for domain subdivision [S1, S2, unofficial].

## How it works
**Inputs:** AMR scalar field data, a specified isovalue range, and a transfer function mapping scalar values to optical properties (color, opacity) [S1, S2, unofficial].
**Outputs:** A rendered 2D image [S1, S2, unofficial].

**Data Structures:**
1. **K-d-tree:** Represents the spatial domain by converting the AMR hierarchy into blocks of constant resolution [S1, S2, unofficial].
2. **Augmented Octree:** Each node in the k-d-tree is augmented with an octree structure [S1, S2, unofficial].
3. **Node Metadata:** Both k-d-tree and octree nodes contain a 32-bit field where each bit represents a continuous range of isovalues [S1, S2, unofficial]. This allows for rapid identification of nodes containing values within the user-specified isovalue range [S1, S2, unofficial].

**Algorithm Steps:**
1. Convert the input AMR hierarchy into a k-d-tree structure where each leaf node represents a block of constant resolution [S1, S2, unofficial].
2. Augment each node of the constructed k-d-tree with an octree [S1, S2, unofficial].
3. Populate a 32-bit field in each k-d-tree and octree node, where each bit corresponds to a specific continuous isovalue range [S1, S2, unofficial].
4. Traverse the k-d-tree and octree structures to identify regions (nodes) that contain scalar values within the specified isovalue range, as determined by the 32-bit metadata fields [S1, S2, unofficial].
5. Render the identified regions back-to-front using hierarchical splatting [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_amr_volume(AMR_data, isovalue_range, transfer_function):
    # Step 1: Convert AMR hierarchy to a k-d-tree structure of constant-resolution blocks
    # The exact conversion procedure for irregular AMR patches is not detailed in the sources [G].
    kdtree = []
    for patch in AMR_data.patches:
        kdtree.append(create_kdtree_node(patch))

    # Step 2: Augment each k-d-tree node with an octree
    for node in kdtree:
        node.octree = build_octree_from_node(node)

    # Step 3: Populate 32-bit isovalue range fields
    # Each bit represents a continuous range of isovalues
    for node in kdtree:
        node.isovalue_mask = 0
        for i in range(32):
            if node.data_contains_range(i, isovalue_range):
                node.isovalue_mask |= (1 << i)

    # Step 4: Identify regions containing values within the specified isovalue range
    isovalue_range_mask = compute_range_mask(isovalue_range)
    relevant_nodes = [node for node in kdtree if node.isovalue_mask & isovalue_range_mask]

    # Step 5: Render identified regions back-to-front using hierarchical splatting
    # This builds on [Splatting](23-splatting.md) and [Octree-based Hierarchy](12129-octree-based-hierarchy.md)
    image = composite_back_to_front(relevant_nodes, transfer_function)
    return image

def compute_range_mask(isovalue_range):
    mask = 0
    for i in range(32):
        if isovalue_range.contains_range(i):
            mask |= (1 << i)
    return mask
```

**Parameters:**
* **Isovalue Range:** The range of scalar values to visualize [S1, S2, unofficial].
* **Transfer Function:** Maps scalar values to color and opacity [S1, S2, unofficial].
* **Bit Field Size:** Fixed at 32 bits per node [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
* **Precision:** The 32-bit field limits the granularity of isovalue range identification to 32 continuous ranges [S1, S2, unofficial].
* **AMR Conversion:** The conversion from AMR to k-d-tree must handle irregular block sizes and refinement levels correctly [G].

## Operating characteristics
* **Time Complexity:** Not given in the sources.
* **Memory Complexity:** Not given in the sources.
* **Performance:** Improves rendering quality considerably compared to slicing-based approaches [S1, unofficial]. Supports a more complex light model with wavelength-dependent absorption [S1, unofficial].
* **Strengths:** Efficient culling of irrelevant data via isovalue range identification; handles AMR data structure naturally through k-d-tree conversion [S1, S2, unofficial].
* **Limitations:** Limited to scalar fields with specified isovalue ranges; 32-bit field constraint on isovalue granularity [S1, S2, unofficial].
* **When to use:** When rendering AMR data with specific isovalue ranges and requiring high-quality volume rendering with complex lighting [S1, S2, unofficial].
* **When not to use:** Not given in the sources.

## Background
The method was introduced by Park, Bajaj & Siddavanahalli in 2002 [S1, S2, unofficial]. It builds on the hierarchical splatting approach by Laur & Hanrahan (1991) [S1, unofficial].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc894061/, fetched 2026-10-26 (unofficial)
[S2] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
[S3] Visualization of Scalar Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc901373/?q=adaptive mesh refinement visualization ray casting, fetched 2026-10-28 (unofficial)
