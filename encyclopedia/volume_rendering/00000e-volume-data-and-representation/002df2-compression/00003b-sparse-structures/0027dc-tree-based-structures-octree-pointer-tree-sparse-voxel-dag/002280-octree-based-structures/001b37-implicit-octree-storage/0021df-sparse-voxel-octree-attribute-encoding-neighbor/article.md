# Sparse Voxel Octree Attribute Encoding (Neighbor)


## Overview
This method encodes the attributes of a sparse voxel octree by storing deltas relative to the attributes of neighboring voxels, rather than storing absolute values. It solves the problem of compressing high-dimensional attribute data (e.g., color, normals) in sparse structures by exploiting spatial coherence. It differs from [Sparse Voxel Octree Attribute Encoding (Parent)](2705-svo-node-encoding.md) by using neighbor-based prediction instead of parent-based prediction, and from [SVO Node Encoding](2705-svo-node-encoding.md) by focusing specifically on attribute delta compression rather than general node bit-packing.

## How it works
The method operates on a set of occupied voxel coordinates and their associated attribute vectors. It assumes the existence of a neighbor lookup mechanism (provided by the parent [Implicit Octree Storage](939-implicit-octree-storage.md) or a hash map) to retrieve the attributes of adjacent voxels.

### Inputs and Outputs
*   **Inputs**:
    *   `coord`: A tensor of shape $[N, 3]$ containing the integer coordinates $(x, y, z)$ of each of the $N$ sparse voxels [S1, unofficial].
    *   `res`: The resolution of the bounding grid, used to determine neighbor offsets [S1, unofficial].
    *   `attr`: A tensor of shape $[N, C]$ containing the original attribute values for each voxel, where $C$ is the number of channels [S1, unofficial].
*   **Outputs**:
    *   `delta`: A tensor of shape $[N, C]$ containing the encoded delta values [S1, unofficial].

### Data Structures
*   **Coordinate Tensor**: $[N, 3]$ integers.
*   **Attribute Tensor**: $[N, C]$ floats or integers.
*   **Delta Tensor**: $[N, C]$ floats or integers.

### Algorithm
The encoding process computes the difference between the current voxel's attribute and a predicted value derived from its neighbors. The decoding process reconstructs the attribute by adding the delta to the predicted value.

1.  **Neighbor Identification**: For each voxel $i$ at `coord[i]`, identify its valid neighbors. In a standard octree context, neighbors are voxels at `coord[i] + offset` where `offset` is a 3D vector with components in $\{-1, 0, 1\}$, excluding $(0,0,0)$. Only neighbors that exist in the sparse set are considered [S1, unofficial].
2.  **Prediction**: Compute the predicted attribute $\hat{A}_i$ for voxel $i$. The specific weighting or selection of neighbors is not detailed in the source, but typically involves averaging or selecting the closest valid neighbor. Let $N_i$ be the set of valid neighbor indices for voxel $i$.
    $$ \hat{A}_i = \text{Predict}( \{ \text{attr}[j] \mid j \in N_i \} ) $$
    [G] If no neighbors exist, $\hat{A}_i = 0$.
3.  **Delta Calculation (Encode)**: Compute the delta for each channel $c$ of voxel $i$:
    $$ \text{delta}[i, c] = \text{attr}[i, c] - \hat{A}_i $$
4.  **Reconstruction (Decode)**: To decode, retrieve the neighbors' attributes and add the delta:
    $$ \text{attr}[i, c] = \hat{A}_i + \text{delta}[i, c] $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_sparse_voxel_octree_attr_neighbor_cpu(coord, res, attr):
    """
    Encodes attributes using neighbor deltas.
    
    Args:
        coord: [N, 3] tensor of voxel coordinates.
        res: Resolution of the grid (used for boundary checks if needed).
        attr: [N, C] tensor of original attributes.
        
    Returns:
        delta: [N, C] tensor of encoded deltas.
    """
    N = coord.shape[0]
    C = attr.shape[1]
    delta = torch.zeros_like(attr)
    
    # Create a lookup map from coordinate to index for neighbor finding
    # [G] Implementation detail: In practice, this is often a hash map or 
    # a sorted array with binary search.
    coord_to_idx = {tuple(coord[i].tolist()): i for i in range(N)}
    
    # Define neighbor offsets (excluding self)
    # [G] Standard 26-neighbor kernel. For 6-neighbor, use only axis-aligned offsets.
    offsets = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                offsets.append((dx, dy, dz))
                
    for i in range(N):
        cx, cy, cz = coord[i]
        neighbor_attrs = []
        
        for dx, dy, dz in offsets:
            nx, ny, nz = cx + dx, cy + dy, cz + dz
            neighbor_idx = coord_to_idx.get((nx, ny, nz))
            if neighbor_idx is not None:
                neighbor_attrs.append(attr[neighbor_idx])
                
        if len(neighbor_attrs) > 0:
            # [G] Simple average prediction. Source does not specify weighting.
            predicted = torch.stack(neighbor_attrs).mean(dim=0)
        else:
            predicted = torch.zeros(C)
            
        delta[i] = attr[i] - predicted
        
    return delta

def decode_sparse_voxel_octree_attr_neighbor_cpu(coord, res, delta, neighbor_attrs_lookup):
    """
    Decodes attributes from neighbor deltas.
    
    Args:
        coord: [N, 3] tensor of voxel coordinates.
        res: Resolution of the grid.
        delta: [N, C] tensor of encoded deltas.
        neighbor_attrs_lookup: A function or map that returns neighbor attributes 
                               given a coordinate.
                               
    Returns:
        attr: [N, C] tensor of reconstructed attributes.
    """
    N = coord.shape[0]
    C = delta.shape[1]
    attr = torch.zeros_like(delta)
    
    offsets = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                offsets.append((dx, dy, dz))
                
    for i in range(N):
        cx, cy, cz = coord[i]
        neighbor_attrs = []
        
        for dx, dy, dz in offsets:
            nx, ny, nz = cx + dx, cy + dy, cz + dz
            # [G] Implementation detail: neighbor_attrs_lookup should return 
            # the attribute tensor for the neighbor at (nx, ny, nz) if it exists.
            neighbor_attr = neighbor_attrs_lookup((nx, ny, nz))
            if neighbor_attr is not None:
                neighbor_attrs.append(neighbor_attr)
                
        if len(neighbor_attrs) > 0:
            predicted = torch.stack(neighbor_attrs).mean(dim=0)
        else:
            predicted = torch.zeros(C)
            
        attr[i] = predicted + delta[i]
        
    return attr
```

### Parameters and Edge Cases
*   **Resolution (`res`)**: Used to define the grid bounds. If a neighbor coordinate falls outside $[0, \text{res}-1]^3$, it is ignored [S1, unofficial].
*   **Empty Neighbors**: If a voxel has no valid neighbors (isolated voxel), the prediction is zero, and the delta equals the original attribute [G].
*   **Precision**: Delta encoding is sensitive to quantization errors if attributes are integer-valued. Floating-point attributes may suffer from precision loss if deltas are quantized [G].

## Subtopics

### Sparse Voxel Octree Attribute Encoding (Parent)
Decodes the attribute of a sparse voxel octree from its parent node and its deltas. It uses the parent's attribute as the prediction basis rather than neighbors.
*   **Inputs**: `octree` (uint8 tensor of structure), `depth` (int), `attr` ([N, C] tensor), `delta` (uint8 tensor) [S1, unofficial].
*   **Difference**: Uses hierarchical parent-child relationship instead of spatial neighbor adjacency. Suitable for hierarchical compression where parent attributes are stable [S1, unofficial].

### Vector to Closest Point (VCP) Octree
A surface representation storing a vector field where each grid point stores a vector to the closest point on the original geometry. It stores direction and distance, allowing normals to be derived from interpolating VCP vectors rather than stored explicitly.
*   **Optimization**: Compresses VCP vector into one 32-bit integer by converting vector into two spherical angles and a radius [S2, unofficial].
*   **Optimization**: Shares VCP vectors between up to eight neighbors by storing them next to each other in memory as a sparse 3D texture [S2, unofficial].
*   **Simplification**: Checks for merging opportunities if multiple leaf nodes have surface data. Compares interpolated vectors from the single parent cell with exact vectors in non-empty leaf cells using Euclidean distance. If max distance < threshold, merges children to a non-empty leaf node at the next level [S2, unofficial].
*   **Memory**: Requires 8 vectors per voxel for the complete grid, compared to 8 scalar values for a scalar field [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Encoding/Decoding is $O(N \cdot K)$, where $N$ is the number of voxels and $K$ is the number of neighbors checked (typically 26 for full 3D neighborhood, or 6 for axis-aligned). Neighbor lookup is $O(1)$ with a hash map or $O(\log N)$ with a sorted array.
*   **Memory Complexity**: Stores $N \times C$ deltas. If deltas are smaller in magnitude than original attributes, they can be quantized to fewer bits, reducing memory.
*   **Strengths**: Exploits spatial coherence, which is often higher than hierarchical coherence in dense regions.
*   **Limitations**: Requires efficient neighbor lookup. Performance degrades if the neighbor structure is sparse or irregular.
*   **When to use**: When spatial coherence is high and the octree structure makes neighbor lookup efficient.
*   **When not to use**: When the attribute changes rapidly between neighbors (e.g., noise), as deltas will be large and compression inefficient.

## Background
The method is part of the `o-voxel-gpu` library, which provides efficient sparse voxel octree operations on GPU and CPU [S1, unofficial]. It is designed for real-time rendering and compression of large-scale 3D data.

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
