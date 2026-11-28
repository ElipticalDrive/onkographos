# Octree Node Decoder

Also called: RocNet Node Decoder

## Overview
The Octree Node Decoder is a recursive neural network component that transforms 4D feature vectors stored in octree nodes into explicit binary occupancy voxel grids or intermediate features [S1]. It distinguishes between interior nodes, which propagate features to children, and leaf nodes, which resolve features into spatial voxels [S1]. It is used within the RocNet architecture to efficiently reconstruct 3D shapes from hierarchical latent representations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D feature vector associated with an octree node.
*   **Output:** For interior nodes, updated feature vectors for the node and its 8 children. For leaf nodes, an explicit binary occupancy voxel grid of size $k \times k \times k$.

**Data Structures:**
*   **Octree Node:** Contains a 4D feature vector and a classification label (Interior, Empty, Full, Mixed) [S1].
*   **Feature Vector:** 4D tensor. The specific channel count is not explicitly defined in the sources, but it is transformed via 3D convolutions [S1].

**Parameters:**
*   **Leaf dimension ($k$):** The output voxel size for leaf nodes. The leaf decoder enlarges the effective voxel size from 4 to $k$ [S1].
*   **Kernel Size:** 4 for transposed convolutions in the leaf decoder [S1].
*   **Stride:** 2 for transposed convolutions in the leaf decoder [S1].
*   **Padding:** 1 for transposed convolutions in the leaf decoder [S1].
*   **Activation Functions:** ELU for intermediate layers; Sigmoid for the final leaf layer [S1].

**Steps:**
1.  **Recursive Decoding:** Start with the root node's 4D feature vector. Apply the node decoder recursively until leaf nodes are reached [S1].
2.  **Interior Node Processing:**
    *   Apply a 3D convolutional layer to the current node's feature vector.
    *   Apply a separate convolutional layer for each of the 8 child nodes.
    *   Apply batch normalization to the outputs [S1].
    *   **Note:** Node decoders for different depths do not share weights [S1].
3.  **Leaf Node Classification Check:**
    *   If the leaf is classified as **Empty** or **Full**, skip decoding [S1].
    *   If the leaf is classified as **Mixed**, proceed to the leaf decoder [S1].
4.  **Leaf Decoder Processing:**
    *   Apply stacked transposed 3D convolutional layers with kernel size 4, stride 2, and padding 1 [S1].
    *   Apply batch normalization and ELU activation after each layer, except the final one [S1].
    *   The final layer uses sigmoid activation without batch normalization [S1].
    *   The output is an explicit binary occupancy voxel grid of size $k \times k \times k$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_octree_node(node, depth):
    """
    Recursively decodes an octree node.
    node: Object containing 'features' (4D tensor) and 'type' (Interior, Empty, Full, Mixed).
    depth: Current depth in the octree tree.
    """
    if node.type == 'Empty' or node.type == 'Full':
        # Skip decoding for efficiency
        return None

    if node.type == 'Interior':
        # Interior node processing
        # Apply 3D convolution to the node itself
        node_features = conv_3d(node.features)
        
        # Apply convolution to each of the 8 children
        # Note: Weights are not shared across depths [S1]
        child_features = []
        for i in range(8):
            child_feat = conv_3d_child(node.features, depth, i)
            child_feat = batch_norm(child_feat)
            child_features.append(child_feat)
        
        # Update node and children features
        node.features = node_features
        for i, child in enumerate(node.children):
            child.features = child_features[i]
            
        # Recurse into children
        for child in node.children:
            decode_octree_node(child, depth + 1)

    elif node.type == 'Mixed':
        # Leaf node processing
        # Apply leaf decoder
        voxel_grid = leaf_decoder(node.features)
        node.voxel_grid = voxel_grid

def leaf_decoder(features):
    """
    Transforms 4D feature vector into k x k x k voxel grid.
    """
    x = features
    # Stacked transposed 3D convolutions
    # Kernel size 4, stride 2, padding 1 [S1]
    x = transposed_conv_3d(x, kernel_size=4, stride=2, padding=1)
    x = batch_norm(x)
    x = elu(x)
    
    # Additional layers as needed to reach dimension k
    # The operation enlarges voxel size from 4 to k [S1]
    # Implementation detail: Number of layers depends on k
    # For example, if k=8, one layer might suffice (4 -> 8)
    # If k=16, two layers (4 -> 8 -> 16)
    
    # Final layer
    x = transposed_conv_3d(x, kernel_size=4, stride=2, padding=1)
    # No batch normalization on final layer [S1]
    x = sigmoid(x)
    
    return x
```

## Subtopics

### [OGN Decoder for 3D Shape Generation](21506-ogn-decoder-for-3d-shape-generation.md)
The Octree Generating Network (OGN) is a deep convolutional decoder architecture that generates volumetric 3D outputs in a compute- and memory-efficient manner by using an octree representation. It learns to predict both the structure of the octree (which cells to subdivide) and the occupancy values of individual cells. Unlike standard decoders acting on regular voxel grids which have cubic complexity, the OGN does not have cubic complexity, allowing representing much higher resolution outputs with a limited memory budget. It maps a latent vector to a hierarchical octree representation of a 3D shape, supporting multiple output resolutions (64^3, 128^3, 256^3, 512^3).

## Operating characteristics
**Time and Memory Complexity:**
*   The method avoids cubic complexity associated with regular voxel grids by only decoding "Mixed" leaves [S1].
*   Memory usage is proportional to the number of non-empty/non-full nodes and leaves [S1].

**Strengths and Limitations:**
*   **Strengths:** Efficient representation of 3D shapes; supports multiple output resolutions; skips decoding for empty/full leaves [S1].
*   **Limitations:** Node decoders for different depths do not share weights, potentially increasing parameter count [S1].

**When to Use:**
*   When reconstructing 3D shapes from hierarchical latent representations with sparse geometry [S1].

**When Not to Use:**
*   When dense, uniform voxel grids are required without hierarchical structure [S1].

## Background
The Octree Node Decoder was introduced in the RocNet (Recursive Octree Network) paper [S1]. It is part of a system that uses a node classifier to recover octree topology during decoding [S1]. The decoder transforms 4D features back to explicit binary occupancy voxel grids [S1].

## Sources
[S1] RocNet: Recursive Octree Network for Efficient 3D Deep Representation, https://doi.org/10.1109/3dv50981.2020.00051, fetched 2026-10-26
