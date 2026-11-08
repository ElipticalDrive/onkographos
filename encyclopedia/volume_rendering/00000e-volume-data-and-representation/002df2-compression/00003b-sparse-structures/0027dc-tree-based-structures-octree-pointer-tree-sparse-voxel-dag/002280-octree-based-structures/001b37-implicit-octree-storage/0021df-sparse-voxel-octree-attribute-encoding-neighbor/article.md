# Sparse Voxel Octree Attribute Encoding (Neighbor)


## Overview
This method encodes the attributes of a sparse voxel octree by storing deltas relative to the attributes of neighboring voxels, rather than storing absolute values or parent-relative deltas. It solves the problem of high memory bandwidth and storage costs for dense attribute data (e.g., color, normals) in sparse representations by exploiting spatial coherence. Unlike its sibling [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md) which handles topology, or [Sparse Voxel Octree Attribute Encoding (Parent)](9709-sparse-voxel-octree-attribute-encoding-parent.md) which uses the parent node, this method uses immediate spatial neighbors to predict the current voxel's attribute.

## How it works
**Inputs:**
- `coord`: An $[N, 3]$ tensor containing the integer coordinates of each of the $N$ sparse voxels.
- `res`: The resolution of the sparse voxel grid (a scalar or vector defining the grid bounds).
- `attr`: An $[N, C]$ tensor containing the original attribute values for each voxel, where $C$ is the number of attribute channels.

**Outputs:**
- `delta`: An $[N, C]$ tensor containing the encoded deltas.

**Data Structures:**
The method assumes a global grid of resolution `res`. Voxels are identified by their integer coordinates $(x, y, z)$. The "neighbors" are typically defined as the 6-connected (face-sharing) or 26-connected (edge/corner-sharing) neighbors within the grid. For implementation, a hash map or a dense grid buffer (if memory permits) is used to look up the attribute of a neighbor given its coordinate.

**Steps:**
1. **Neighbor Lookup:** For each voxel $i$ at coordinate $p_i = (x_i, y_i, z_i)$, identify its set of existing neighbors $\mathcal{N}_i$. A neighbor $j$ exists if $p_j$ is in the set of active voxel coordinates and $p_j$ is adjacent to $p_i$ (e.g., $|x_i - x_j| + |y_i - y_j| + |z_i - z_j| = 1$ for face neighbors).
2. **Prediction:** Select a reference neighbor $j^* \in \mathcal{N}_i$. If multiple neighbors exist, a common strategy is to pick the one with the smallest coordinate index or the one with the most significant bit difference, but the source does not specify a deterministic tie-breaking rule [S1, unofficial]. If no neighbors exist, the prediction is zero.
3. **Delta Calculation:** Compute the delta for voxel $i$ as the difference between its attribute and the neighbor's attribute:
   $$ \delta_i = \text{attr}_i - \text{attr}_{j^*} $$
   where $\text{attr}_i$ is the $C$-dimensional attribute vector of voxel $i$.
4. **Output:** Store $\delta_i$ in the output tensor.

**Parameters:**
- `coord`: $[N, 3]$ tensor.
- `res`: Grid resolution.
- `attr`: $[N, C]$ tensor.
- `delta`: $[N, C]$ tensor.

**Edge Cases:**
- **Boundary Voxels:** Voxels on the edge of the grid may have fewer than 6 neighbors. The method must handle missing neighbors gracefully (e.g., by skipping them or using a default prediction of 0).
- **No Neighbors:** If a voxel has no active neighbors, the delta is simply the attribute value itself (assuming a zero prediction).
- **Precision:** The delta values should be quantized or stored in a format suitable for compression, as they are expected to be small if spatial coherence is high.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def encode_sparse_voxel_octree_attr_neighbor_cpu(
    coord: torch.Tensor,
    res: int,
    attr: torch.Tensor
) -> torch.Tensor:
    """
    Encode sparse voxel octree attributes using neighbor deltas.
    
    Args:
        coord: [N, 3] tensor of voxel coordinates.
        res: Resolution of the grid.
        attr: [N, C] tensor of voxel attributes.
        
    Returns:
        delta: [N, C] tensor of deltas relative to neighbors.
    """
    N = coord.shape[0]
    C = attr.shape[1]
    delta = torch.zeros_like(attr)
    
    # Create a hash map for fast neighbor lookup: coord -> index in attr
    # In a real implementation, this might be a dense grid or a hash table.
    # Here we use a simple dictionary for clarity.
    coord_to_idx = {tuple(coord[i].tolist()): i for i in range(N)}
    
    # Define neighbor offsets (6-connected)
    neighbors_offsets = [
        (1, 0, 0), (-1, 0, 0),
        (0, 1, 0), (0, -1, 0),
        (0, 0, 1), (0, 0, -1)
    ]
    
    for i in range(N):
        p = coord[i].tolist()
        current_attr = attr[i]
        found_neighbor = False
        
        # Try to find a neighbor
        for dx, dy, dz in neighbors_offsets:
            neighbor_coord = (p[0] + dx, p[1] + dy, p[2] + dz)
            if neighbor_coord in coord_to_idx:
                neighbor_idx = coord_to_idx[neighbor_coord]
                neighbor_attr = attr[neighbor_idx]
                delta[i] = current_attr - neighbor_attr
                found_neighbor = True
                break # Use the first found neighbor
        
        if not found_neighbor:
            # No neighbor found, delta is the attribute itself (prediction 0)
            delta[i] = current_attr
            
    return delta

def decode_sparse_voxel_octree_attr_neighbor_cpu(
    coord: torch.Tensor,
    res: int,
    delta: torch.Tensor
) -> torch.Tensor:
    """
    Decode sparse voxel octree attributes from neighbor deltas.
    
    Args:
        coord: [N, 3] tensor of voxel coordinates.
        res: Resolution of the grid.
        delta: [N, C] tensor of deltas.
        
    Returns:
        attr: [N, C] tensor of reconstructed attributes.
    """
    N = coord.shape[0]
    C = delta.shape[1]
    attr = torch.zeros_like(delta)
    
    # Create a hash map for fast neighbor lookup
    coord_to_idx = {tuple(coord[i].tolist()): i for i in range(N)}
    
    neighbors_offsets = [
        (1, 0, 0), (-1, 0, 0),
        (0, 1, 0), (0, -1, 0),
        (0, 0, 1), (0, 0, -1)
    ]
    
    for i in range(N):
        p = coord[i].tolist()
        current_delta = delta[i]
        found_neighbor = False
        
        for dx, dy, dz in neighbors_offsets:
            neighbor_coord = (p[0] + dx, p[1] + dy, p[2] + dz)
            if neighbor_coord in coord_to_idx:
                neighbor_idx = coord_to_idx[neighbor_coord]
                neighbor_attr = attr[neighbor_idx]
                attr[i] = neighbor_attr + current_delta
                found_neighbor = True
                break
        
        if not found_neighbor:
            # No neighbor found, attribute is the delta itself
            attr[i] = current_delta
            
    return attr
```

## Subtopics

### Sparse Voxel Octree Attribute Encoding (Parent)
Decodes the attribute of a sparse voxel octree from its parent node and its deltas. It uses the parent node's attribute as the prediction source instead of spatial neighbors. This is useful when the octree structure is explicitly available and parent-child relationships are well-defined. It is chosen when the octree hierarchy is deep and parent-child coherence is stronger than spatial neighbor coherence.

## Operating characteristics
- **Time Complexity:** $O(N \cdot k)$ where $N$ is the number of voxels and $k$ is the number of neighbors checked (typically 6 or 26). Neighbor lookup is $O(1)$ with a hash map or $O(\log N)$ with a tree.
- **Memory Complexity:** $O(N \cdot C)$ for storing the deltas. The hash map adds overhead.
- **Strengths:** Exploits spatial coherence, which is often high in natural scenes. Can achieve better compression ratios than parent-based encoding if the octree is shallow or irregular.
- **Limitations:** Requires neighbor lookups, which can be cache-unfriendly if the voxel coordinates are not sorted. Boundary conditions must be handled carefully.
- **When to use:** When spatial coherence is high and the octree structure is not strictly hierarchical or when parent-child relationships are not the primary source of redundancy.

## Background
This method is part of the [o-voxel-gpu](https://github.com/quantaji/o-voxel-gpu) library, which provides efficient GPU-accelerated operations for sparse voxel octrees. It is designed for real-time rendering and compression of 3D data.

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
