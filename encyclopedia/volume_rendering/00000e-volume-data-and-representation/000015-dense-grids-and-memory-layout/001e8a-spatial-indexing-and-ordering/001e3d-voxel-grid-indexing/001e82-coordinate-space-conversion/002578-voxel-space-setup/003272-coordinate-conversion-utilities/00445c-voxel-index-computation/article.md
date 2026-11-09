# Voxel Index Computation

Also called: Voxel Grid Indexing, Discrete Coordinate Mapping

## Overview
Computes discrete voxel indices for a batch of 3D points by mapping their continuous coordinates into a structured grid defined by bounding boxes and voxel dimensions [S1, unofficial]. It differs from [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md) by operating on a unified 3D grid structure rather than slicing the volume into 2D planes, making it suitable for dense volumetric queries and ray marching initialization [S1, unofficial].

## How it works
**Inputs:**
- `points`: Tensor of shape $(N, 3)$ containing 3D coordinates.
- `bounding_boxes`: Tensor of shape $(B, 2, 3)$ or $(B, 3)$ defining the minimum and maximum bounds for each batch item.
- `batch_ids`: Tensor of shape $(N)$ mapping each point to a batch item.
- `voxel_size`: Scalar or Tensor defining the grid resolution.
- `shift`: Boolean flag for coordinate alignment.
- `cat_batch_ids`: Boolean flag for output format.

**Outputs:**
- `voxel_indices`: Tensor of shape $(N, 3)$ if `cat_batch_ids` is false, or $(N, 4)$ if true (concatenated batch ID).

**Data Structures:**
- Voxel indices are stored as integers (long).
- Bounding boxes are typically stored as $(min, max)$ pairs.

**Steps:**
1. **Batch Selection**: Retrieve the bounding box minimums for each point based on `batch_ids`.
   $$ bb\_mins = bounding\_boxes[batch\_ids][:, 0] $$
2. **Voxel Size Retrieval**: Handle scalar or tensor voxel sizes. If scalar, broadcast to all points. If tensor, index by `batch_ids`.
   $$ voxel\_size\_b = \begin{cases} voxel\_size & \text{if scalar} \\ voxel\_size[batch\_ids] & \text{if tensor} \end{cases} $$
3. **Dimension Expansion**: If `voxel_size_b` is 1D (shape $(N)$ or $(B)$), expand to 2D (shape $(N, 1)$ or $(B, 1)$) for broadcasting with 3D coordinates.
   $$ voxel\_size\_b = voxel\_size\_b.unsqueeze(-1) $$
4. **Shift Adjustment**: If `shift` is true, adjust the bounding box minimums to center the voxel grid.
   $$ bb\_mins = bb\_mins - 0.5 \times voxel\_size\_b $$
5. **Index Calculation**: Compute the integer voxel indices using floor division.
   $$ voxel\_indices = \lfloor \frac{points - bb\_mins}{voxel\_size\_b} \rfloor $$
   Convert to long integers.
6. **Batch ID Concatenation**: If `cat_batch_ids` is true, concatenate `batch_ids` to `voxel_indices` along dimension 1.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_voxel_indices(points, bounding_boxes, batch_ids, voxel_size, shift=False, cat_batch_ids=False):
    # Step 1: Select bounding box minimums
    # bounding_boxes shape: (B, 2, 3) or (B, 3)
    # Assuming (B, 2, 3) format where index 0 is min, 1 is max
    if bounding_boxes.dim() == 3:
        bb_mins = bounding_boxes[batch_ids][:, 0]
    else:
        # Fallback for (B, 3) format if min is the only bound provided or interpreted differently
        # Note: Source implies (B, 2, 3) structure for min/max
        bb_mins = bounding_boxes[batch_ids]

    # Step 2: Retrieve voxel size
    if isinstance(voxel_size, torch.Tensor):
        voxel_size_b = voxel_size[batch_ids]
    else:
        voxel_size_b = torch.full_like(points, voxel_size)

    # Step 3: Expand dimensions for broadcasting
    if voxel_size_b.dim() == 1:
        voxel_size_b = voxel_size_b.unsqueeze(-1)

    # Step 4: Shift adjustment
    if shift:
        bb_mins = bb_mins - 0.5 * voxel_size_b

    # Step 5: Compute indices
    # points shape: (N, 3)
    # bb_mins shape: (N, 3)
    # voxel_size_b shape: (N, 1) or (N, 3)
    voxel_indices = torch.floor((points - bb_mins) / voxel_size_b).long()

    # Step 6: Concatenate batch IDs
    if cat_batch_ids:
        batch_ids_expanded = batch_ids.unsqueeze(-1).long()
        voxel_indices = torch.cat([batch_ids_expanded, voxel_indices], dim=1)

    return voxel_indices
```

**Edge Cases:**
- **Precision**: Floating-point errors in division may cause off-by-one errors at boundaries. Using `floor` is standard but may require epsilon adjustments for exact boundary alignment [G].
- **Out of Bounds**: Points outside the bounding box will result in negative or excessively large indices. Clamping is not performed by this method [G].
- **Zero Voxel Size**: Division by zero if `voxel_size` is 0. Not handled in source [S1, unofficial].

## Subtopics

### [Lighting Fast Querier](11265-lighting-fast-querier.md)
A class that manages the querying of 3D points from a point cloud for ray marching, handling coordinate transformations, grid indexing via CUDA, and ray generation. It uses Voxel Index Computation for efficient spatial lookups.

## Operating characteristics
- **Time Complexity**: $O(N)$ for $N$ points, as it involves element-wise operations and indexing.
- **Memory Complexity**: $O(N)$ to store the output indices.
- **Performance**: Efficient on GPU due to tensor operations. Performance scales with batch size and number of points.
- **Strengths**: Simple, fast, and differentiable (if using soft floor or specific implementations).
- **Limitations**: Does not handle out-of-bounds points. Requires pre-computed bounding boxes.

## Background
Originated in the context of 3D Gaussian Splatting and volumetric rendering, specifically in the VGVGGT framework [S1, unofficial]. It is used to map continuous 3D points to discrete voxel grids for efficient rendering and querying.

## Sources
[S1] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
