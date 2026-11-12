# SVO Attribute Compression via Parent Delta Encoding

Also called: Parent Delta Encoding

## Overview
SVO Attribute Compression via Parent Delta Encoding is a method for compressing volumetric attributes (such as color or alpha) stored in Sparse Voxel Octrees (SVOs) by encoding deltas relative to parent node attributes rather than direct spatial neighbors [S1]. It solves the problem of random access complexity and missing neighbors in 3D sparse data, where direct neighbor access requires logarithmic time and potentially loading the entire SVO into memory, unlike the constant-time access available in 2D images [S1]. It sets itself apart from sibling methods like Neighbor-based encoding by leveraging the hierarchical structure of the SVO to ensure streamability and low memory complexity during decoding [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A Sparse Voxel Octree (SVO) where leaf nodes contain attribute values (e.g., 24-bit RGB color) and internal nodes represent spatial subdivisions without inherent attribute values [S1].
*   **Output:** A compressed bitstream of attribute deltas, typically passed to an entropy coder [S1].

**Data Structures**
The SVO structure consists of:
*   **Leaf Nodes:** Represent voxels. These hold the original attribute values (e.g., color). Geometry existence is typically stored separately (e.g., 1 bit) [S1].
*   **Internal Nodes:** Represent subdivisions of space. They do not hold attribute values initially but are populated during the propagation step [S1].

**Steps**
1.  **Attribute Propagation (Mipmapping):** Propagate attribute values from child leaf nodes to their parent nodes. For each internal node, compute its attribute value as a function of its children's attributes. Common aggregation functions include minimum, maximum, or average [S1].
    *   *Note:* This step increases the initial model size because parent nodes must store aggregated values, but it prepares the data for delta encoding [S1].
2.  **Delta Encoding:** For each leaf node, compute the delta between its original attribute value and the attribute value of its parent node (computed in Step 1).
    *   Formula: $\Delta = \text{ChildAttribute} - \text{ParentAttribute}$
    *   The parent attribute is located half a voxel away from the child's center, ensuring attributes are typically similar, resulting in small deltas [S1].
3.  **Entropy Coding:** Pass the sequence of computed deltas to an entropy coder (e.g., DEFLATE/zlib) to produce the final compressed stream [S1]. The entropy coder is treated as a black box and can be swapped out [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_parent_attribute(children_attributes, aggregation_mode):
    """
    Computes the parent attribute from its children.
    aggregation_mode: 'min', 'max', or 'average'
    """
    if aggregation_mode == 'min':
        return min(children_attributes)
    elif aggregation_mode == 'max':
        return max(children_attributes)
    elif aggregation_mode == 'average':
        return sum(children_attributes) / len(children_attributes)
    else:
        raise ValueError("Unsupported aggregation mode")

def encode_svo_attributes(svo_tree):
    """
    Encodes SVO attributes using Parent Delta Encoding.
    Assumes svo_tree is a structure where leaf nodes have 'attribute'
    and internal nodes have 'children' (list of nodes).
    Returns a list of deltas.
    """
    deltas = []
    
    # Step 1: Propagate attributes upwards (Mipmapping)
    # We need to process from leaves up to the root.
    # In a recursive structure, we can compute parent values after children are processed.
    
    def propagate(node):
        if node.is_leaf():
            # Leaf nodes have the original attribute
            return node.attribute
        else:
            # Internal node: aggregate children's attributes
            child_attributes = [propagate(child) for child in node.children]
            # Store the computed parent attribute in the node for later use
            node.computed_attribute = compute_parent_attribute(child_attributes, 'average')
            return node.computed_attribute

    # First pass: Propagate values up the tree
    # The root's computed attribute is not strictly needed for encoding leaves,
    # but the propagation ensures all internal nodes have values.
    propagate(svo_tree.root)

    # Step 2: Encode deltas for leaf nodes
    def encode_leaf(node):
        if node.is_leaf():
            parent_attribute = node.parent.computed_attribute
            delta = node.attribute - parent_attribute
            deltas.append(delta)
        else:
            for child in node.children:
                encode_leaf(child)

    encode_leaf(svo_tree.root)
    
    # Step 3: Entropy coding is handled externally
    # return deltas
    return deltas
```

**Parameters**
*   **Parent Aggregation Function:** Determines how parent attributes are derived from children. Options include minimum, maximum, or average [S1].
*   **Entropy Coder:** The method allows any entropy coder (e.g., zlib/DEFLATE) to be used as a black box [S1].

**Edge Cases and Pitfalls**
*   **Initial Size Increase:** The mipmapping propagation step initially increases the model size because parent nodes must store aggregated values. A net benefit is only realized if the children's values are similar enough that the resulting deltas are small [S1].
*   **Missing Neighbors:** Direct neighbor encoding fails in SVOs because voxels can be surrounded by empty space (missing neighbors) and random access to neighbors has logarithmic complexity [S1]. Parent delta encoding resolves this as every node (except the root) has a parent, and parent access is efficient [S1].

## Operating characteristics
*   **Time Complexity:** Encoding and decoding have low time complexity compared to direct neighbor access in SVOs, which has logarithmic complexity [S1]. Parent access is described as "very easy" and resolves space/time/memory requirements [S1].
*   **Memory Complexity:** The method supports streamability and does not require reading the entire SVO into memory during decoding, unlike direct neighbor access which would require full memory loading [S1].
*   **Compression Ratio:** High compression ratio is achieved when attribute values (like color) are similar across spatially adjacent voxels, resulting in small deltas [S1].
*   **Strengths:** Good SVO interoperability, low memory/time complexity, streamability, and high compression ratio for typical models [S1].
*   **Limitations:** The initial mipmapping step increases model size temporarily; effectiveness depends on attribute similarity [S1].

## Background
The method was developed to address the limitations of extending 2D image compression techniques (like PNG filtering) to 3D volumetric data stored in SVOs. In 2D images, direct spatial neighbors (left, top, top-left) are easily accessible, but in 3D SVOs, random access to neighbors is complex and memory-intensive [S1]. The approach was inspired by the need for good SVO interoperability and streamability while exploiting redundancies in typical voxel models [S1]. It is used in the context of voxel model compression, specifically for attributes like color which comprise the vast majority of information bytes compared to geometry (1 bit vs 24 bits) [S1].

## Sources
[S1] SVO Attribute Compression, https://eisenwave.github.io/voxel-compression-docs/attribute_compression.html, fetched 2026-10-27
