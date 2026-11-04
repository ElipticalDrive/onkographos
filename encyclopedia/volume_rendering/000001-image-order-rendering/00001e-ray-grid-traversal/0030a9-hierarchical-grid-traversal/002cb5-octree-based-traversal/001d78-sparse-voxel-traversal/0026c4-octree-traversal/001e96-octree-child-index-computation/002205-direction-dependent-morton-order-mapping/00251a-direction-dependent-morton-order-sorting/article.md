# Direction-dependent Morton Order Sorting

Also called: Ray sign bit sorting, Sparse Voxel Rasterization Sorting

## Overview
This method sorts sparse voxels into a near-to-far order for each image tile by replacing primitive z-depth with a direction-dependent Morton order, ensuring correct rendering order for all ray directions [S1]. Unlike [3D Gaussian Splatting](3d-gaussian-splatting.md) which duplicates primitives per tile and sorts by z-depth, this technique duplicates voxels per ray sign bit pattern and sorts by Morton order to handle arbitrary viewing angles without popping artifacts [S1].

## How it works
**Inputs:**
- A list of voxels, each with an Octree Morton code (`octpath`) and a unique 1D array index (`voxel_id`) [S1].
- Ray directions (`rd`) for the current frame or tile [S1].

**Outputs:**
- A sorted list of key-value pairs ready for rasterization [S1].

**Data Structures:**
The sorting key-value pair is packed into 64-bit and 32-bit unsigned integers respectively [S1]:
- **Key (64-bit):**
  - Bits 0–15: Tile ID (16 bits) [S1].
  - Bits 16–63: Direction-dependent Morton order (48 bits, where $3L = 48$ and $L=16$) [S1].
- **Value (32-bit):**
  - Bits 0–2: Ray sign bits (3 bits) [S1].
  - Bits 3–31: Voxel ID (29 bits) [S1].

**Parameters:**
- $L=16$: Maximum number of Octree levels [S1].
- Maximum tiles: $2^{16} = 65,536$ [S1].
- Maximum image resolution: $4096 \times 4096$ with $16 \times 16$ tile size [S1].
- Maximum grid resolution: $(2^{16})^3 = 65,536^3$ [S1].
- Maximum voxels: $2^{29} \approx 500$ million [S1].

**Steps:**
1. **Compute Ray Sign Bits:** For each ray direction $\mathbf{rd} = (rd_x, rd_y, rd_z)$, compute the 3-bit sign mask:
   $$ \text{signbits} = 4 \cdot (rd_x < 0) + 2 \cdot (rd_y < 0) + (rd_z < 0) $$
   [S1]

2. **Compute Direction-Dependent Morton Order:** For each voxel with Octree Morton code `octpath` and a specific `signbits` pattern, remap the Morton code using a lookup table `order_tables` indexed by `signbits`. The table permutes the 3-bit child indices at each level to align with the near-to-far order for that ray direction [S1]:
   $$ \text{order} = \sum_{i=0}^{L-1} \text{order\_tables}[\text{signbits}][\text{octpath} \ \& \ 0\text{b}111] \ll 3i $$
   [S1]

3. **Duplicate Voxels:** Duplicate each voxel for each distinct ray sign bit pattern it covers. Attach the corresponding `signbits` to each duplicate [S1].

4. **Sort:** Sort the list of key-value pairs using a GPU Radix sort (or similar stable sort) based on the 64-bit key. The sort order is primarily by Tile ID (MSB), then by Direction-dependent Morton order (LSB) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
MAX_NUM_LEVELS = 16
order_tables = [
    [0, 1, 2, 3, 4, 5, 6, 7],
    [1, 0, 3, 2, 5, 4, 7, 6],
    [2, 3, 0, 1, 6, 7, 4, 5],
    [3, 2, 1, 0, 7, 6, 5, 4],
    [4, 5, 6, 7, 0, 1, 2, 3],
    [5, 4, 7, 6, 1, 0, 3, 2],
    [6, 7, 4, 5, 2, 3, 0, 1],
    [7, 6, 5, 4, 3, 2, 1, 0]
]

def compute_signbits(rd):
    """
    rd: tuple or list of 3 floats (ray direction x, y, z)
    Returns: integer 0-7 representing sign bits (x, y, z)
    """
    return 4 * (rd[0] < 0) + 2 * (rd[1] < 0) + (rd[2] < 0)

def compute_dir_dep_morton_order(octpath, signbits):
    """
    octpath: integer, the standard Octree Morton code
    signbits: integer 0-7, the ray sign bits
    Returns: integer, the direction-dependent Morton order
    """
    table = order_tables[signbits]
    order = 0
    for i in range(MAX_NUM_LEVELS):
        # Extract 3 bits for current level
        child_idx = octpath & 0b111
        # Remap using table
        order |= table[child_idx] << (3 * i)
        # Shift to next level
        octpath = octpath >> 3
    return order

def prepare_sorting_key_value_pairs(voxels, ray_directions):
    """
    voxels: list of dicts with 'octpath' and 'voxel_id'
    ray_directions: list of ray directions for the current tile/frame
    Returns: list of (key_64bit, value_32bit) tuples
    """
    results = []
    for rd in ray_directions:
        signbits = compute_signbits(rd)
        for voxel in voxels:
            # Compute the specific Morton order for this ray direction
            order = compute_dir_dep_morton_order(voxel['octpath'], signbits)
            
            # Pack Key: 16-bit tile_id (assumed 0 for global sort or handled externally) 
            # + 48-bit Morton order. 
            # Note: In practice, tile_id is assigned during rasterization traversal.
            # Here we assume a placeholder tile_id of 0 for the sorting key structure.
            tile_id = 0 
            key = (tile_id << 48) | order
            
            # Pack Value: 3-bit signbits + 29-bit voxel_id
            value = (signbits << 29) | voxel['voxel_id']
            
            results.append((key, value))
            
    # Sort by key (ascending)
    results.sort(key=lambda x: x[0])
    return results
```

**Edge Cases and Pitfalls:**
- The `voxel_id` in the value field must be the 1D array index where the voxel is stored, not the 3D grid $(i, j, k)$ index [S1].
- The maximum number of voxels is limited to $2^{29}$ due to the 29-bit field in the value [S1].
- If the grid resolution exceeds $65,536^3$, custom data types with extra bits are required for the GPU Radix sort [S1].

## Operating characteristics
- **Time Complexity:** Sorting complexity depends on the number of duplicated voxels. With up to $2^{29}$ voxels and 65,536 tiles, the sort is performed on a large list of key-value pairs [S1].
- **Memory Complexity:** Requires storing duplicated voxels. Each voxel is duplicated for each ray sign bit pattern it covers, increasing memory usage compared to non-duplicated approaches [S1].
- **Strengths:** Ensures correct near-to-far rendering order for all ray directions, eliminating popping artifacts common in z-depth sorting for arbitrary viewpoints [S1].
- **Limitations:** Increased memory footprint due to voxel duplication; limited by 64-bit key and 32-bit value constraints unless custom types are used [S1].
- **When to use:** When rendering sparse voxel grids with arbitrary camera orientations where z-depth sorting fails to maintain correct occlusion order [S1].

## Background
This method is part of the Sparse Voxels Rasterization pipeline, introduced in "Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering" [S1]. It addresses the limitations of 3D Gaussian Splatting's z-depth sorting, which can cause artifacts when rays are not aligned with the principal axis [S1]. The technique builds on [Direction-dependent Morton Order Mapping](2662-direction-dependent-morton-order-mapping.md) to remap Octree codes based on ray direction signs [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
