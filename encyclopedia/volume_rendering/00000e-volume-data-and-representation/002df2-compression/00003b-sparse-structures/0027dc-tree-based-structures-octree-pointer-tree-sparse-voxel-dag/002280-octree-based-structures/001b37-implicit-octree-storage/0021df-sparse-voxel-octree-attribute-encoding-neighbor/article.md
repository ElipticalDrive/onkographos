# Sparse Voxel Octree Attribute Encoding (Neighbor)
## Overview
This method encodes the attributes of a sparse voxel octree into deltas from its neighbors, and decodes them back from the neighbors and deltas [S1, unofficial]. It sets itself apart from [Sparse Voxel Octree Attribute Encoding (Parent)](9709-sparse-voxel-octree-attribute-encoding-parent.md) by using immediate spatial neighbors rather than parent nodes for prediction [S1, unofficial].
## How it works
**Inputs:**
- `coord`: An $[N, 3]$ tensor containing the coordinates of each sparse voxel [S1, unofficial].
- `res`: The resolution of the sparse voxel grid [S1, unofficial].
- `attr`: An $[N, C]$ tensor containing the attribute of each sparse voxel [S1, unofficial].

**Outputs:**
- `delta`: An $[N, C]$ tensor containing the deltas [S1, unofficial].

**Data Structures:**
The method operates on the specified tensor shapes. Voxels are identified by their integer coordinates in `coord` [S1, unofficial]. Neighbor lookups require mapping coordinates to tensor indices, typically implemented with a hash map or dense grid buffer [G].

**Steps:**
1. For each voxel $i$ at coordinate $p_i$, identify a neighbor voxel $j$ at coordinate $p_j$ [S1, unofficial].
2. Compute the delta $\delta_i = \text{attr}_i - \text{attr}_j$ [S1, unofficial].
3. Store $\delta_i$ in the output tensor [S1, unofficial].

The source does not specify a deterministic neighbor selection strategy, boundary handling for missing neighbors, or precision requirements for the deltas [S1, unofficial].

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
    """
    N = coord.shape[0]
    C = attr.shape[1]
    delta = torch.zeros_like(attr)
    coord_to_idx = {tuple(coord[i].tolist()): i for i in range(N)}
    # Source does not specify neighbor selection or boundary handling [S1, unofficial]
    neighbor_offsets = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
    for i in range(N):
        p = coord[i].tolist()
        found = False
        for dx, dy, dz in neighbor_offsets:
            nc = (p[0]+dx, p[1]+dy, p[2]+dz)
            if nc in coord_to_idx:
                delta[i] = attr[i] - attr[coord_to_idx[nc]]
                found = True
                break
        if not found:
            delta[i] = attr[i]
    return delta

def decode_sparse_voxel_octree_attr_neighbor_cpu(
    coord: torch.Tensor,
    res: int,
    delta: torch.Tensor
) -> torch.Tensor:
    """
    Decode sparse voxel octree attributes from neighbor deltas.
    """
    N = coord.shape[0]
    C = delta.shape[1]
    attr = torch.zeros_like(delta)
    coord_to_idx = {tuple(coord[i].tolist()): i for i in range(N)}
    neighbor_offsets = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
    for i in range(N):
        p = coord[i].tolist()
        found = False
        for dx, dy, dz in neighbor_offsets:
            nc = (p[0]+dx, p[1]+dy, p[2]+dz)
            if nc in coord_to_idx:
                attr[i] = attr[coord_to_idx[nc]] + delta[i]
                found = True
                break
        if not found:
            attr[i] = delta[i]
    return attr
```

**Parameters:**
- `coord`: $[N, 3]$ tensor [S1, unofficial].
- `res`: Grid resolution [S1, unofficial].
- `attr`: $[N, C]$ tensor [S1, unofficial].
- `delta`: $[N, C]$ tensor [S1, unofficial].

**Edge Cases:**
The source does not specify how to handle boundary voxels, voxels with no active neighbors, or precision requirements for the deltas [S1, unofficial].

## Subtopics

### Sparse Voxel Octree Attribute Encoding (Parent)
Decodes the attribute of a sparse voxel octree from its parent node and its deltas [S1, unofficial]. It uses the parent node's attribute as the prediction source instead of spatial neighbors [S1, unofficial]. Parameters include `octree` (uint8 tensor), `depth` (int), `attr` ($[N, C]$ tensor), and `delta` (uint8 tensor) [S1, unofficial].

### [Vector to Closest Point (VCP) Octree](9709-vector-to-closest-point-vcp-octree.md)
The Vector to Closest Point (VCP) Octree is a hierarchical sparse data structure for surface ray-casting that stores a vector field pointing to the closest point on the original geometry at each grid node, rather than a scalar distance. It solves the problem of block artifacts and loss of sharp features in standard scalar voxel octrees by encoding directionality, allowing for smooth surface approximation and normal derivation via interpolation. It sets itself apart from its parent [Sparse Voxel Octree Attribute Encoding (Neighbor)](2641-sparse-voxel-octree-attribute-encoding-neighbor.md) by using geometric vectors to the surface rather than attribute deltas, and from standard Signed Distance Fields (SDFs) by storing explicit direction information which prevents feature blurring during linear interpolation.

## Operating characteristics
Not given in the sources.

## Background
This method originates from the o-voxel-gpu library [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
