# Boundary Sharing for Adaptive-Resolution Blocks

Also called: Block Boundary Sharing, Slice Sharing for Blocks

## Overview
Boundary Sharing for Adaptive-Resolution Blocks is a technique to eliminate visual artifacts (visible seams or black strips) in parallel volume rendering when volume data is partitioned into 3D blocks rather than 2D slices [S1, unofficial]. It solves the problem of lost ray samples at subvolume boundaries by sharing boundary slices between adjacent blocks [S1, unofficial][S2]. Unlike standard slice-based sharing which only handles 2D cuts, this method manages 3D block interfaces and adapts to multi-resolution data where adjacent blocks may have different voxel densities [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A volume dataset partitioned into $N$ 3D blocks. Each block $B_i$ has a defined spatial extent, resolution (voxel spacing or grid density), and adjacency relationships within a global grid [S1, unofficial].
- **Output:** A single composited image with no boundary artifacts [S1, unofficial].

**Data Structures:**
- **Block:** Defined by its 3D grid dimensions $(D_x, D_y, D_z)$, resolution, and a 3D array of voxel values [S1, unofficial].
- **Shared Slice:** A 2D plane of voxels extracted from a block's boundary face [S1, unofficial].

**Algorithm Steps:**

1. **Identify Boundary Sides:** For each block, determine which of its 6 faces (left, right, front, back, top, bottom) coincide with the global data boundaries. These faces are ignored for sharing. All internal faces (adjacent to another block) require sharing [S1, unofficial].
2. **Determine Sharing Requirements:** A central block shares 6 slices (one per face) [S1, unofficial]. A corner or edge block shares only the slices corresponding to its internal faces [S1, unofficial].
3. **Adapt Resolution for Shared Slices:** If the dataset has multiple resolutions, expand the resolution of a shared slice if it is shared by a higher resolution block, or contract it if shared by a lower resolution block, to maintain consistent resolution within each block [S1, unofficial]. The same physical slice may exist at different resolutions for different adjacent blocks [S1, unofficial].
4. **Render Partial Images:** Each block renders its volume data including its own internal voxels and the shared slices from adjacent blocks, producing a partial image free of boundary artifacts [S1, unofficial].
5. **Recursive Compositing:** Composite the partial images in pairs recursively until a single final image is generated [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_blocks(blocks, grid_dims):
    """
    blocks: list of dicts {'id': int, 'resolution': float, 'voxels': 3D array}
    grid_dims: tuple (Nx, Ny, Nz) defining the global block grid layout
    """
    partial_images = {}
    # Map block IDs to grid coordinates (x, y, z)
    grid_coords = {}
    for b in blocks:
        idx = b['id']
        grid_coords[b['id']] = (idx % grid_dims[0], (idx // grid_dims[0]) % grid_dims[1], (idx // (grid_dims[0] * grid_dims[1])) % grid_dims[2])

    for block in blocks:
        shared_slices = {}
        cx, cy, cz = grid_coords[block['id']]

        # Check all 6 faces
        for dx, dy, dz, face_name in [(-1,0,0,'min_x'), (1,0,0,'max_x'), (0,-1,0,'min_y'), (0,1,0,'max_y'), (0,0,-1,'min_z'), (0,0,1,'max_z')]:
            nx, ny, nz = cx + dx, cy + dy, cz + dz
            # Ignore faces coinciding with global data boundaries
            if not (0 <= nx < grid_dims[0] and 0 <= ny < grid_dims[1] and 0 <= nz < grid_dims[2]):
                continue

            # Find neighbor block
            neighbor_id = nx + ny * grid_dims[0] + nz * grid_dims[0] * grid_dims[1]
            neighbor = next(b for b in blocks if b['id'] == neighbor_id)

            # Extract boundary slice from current block
            if face_name == 'min_x': slice_data = block['voxels'][0, :, :]
            elif face_name == 'max_x': slice_data = block['voxels'][-1, :, :]
            elif face_name == 'min_y': slice_data = block['voxels'][:, 0, :]
            elif face_name == 'max_y': slice_data = block['voxels'][:, -1, :]
            elif face_name == 'min_z': slice_data = block['voxels'][:, :, 0]
            else: slice_data = block['voxels'][:, :, -1]

            # Adapt resolution for adaptive-resolution datasets
            if neighbor['resolution'] > block['resolution']:
                slice_data = upsample_2d(slice_data, block['resolution'], neighbor['resolution'])
            elif neighbor['resolution'] < block['resolution']:
                slice_data = downsample_2d(slice_data, block['resolution'], neighbor['resolution'])

            shared_slices[face_name] = slice_data

        # Render block with internal voxels and shared boundary slices
        # Standard raycasting loop over image pixels using volume and shared slices [G]
        image = raycast_volume(block['voxels'], shared_slices)
        partial_images[block['id']] = image

    # Composite partial images recursively in pairs
    return composite_recursive(partial_images, grid_dims)

def upsample_2d(data, src_res, dst_res):
    # Interpolate 2D data to higher resolution [G]
    return data

def downsample_2d(data, src_res, dst_res):
    # Average 2D data to lower resolution [G]
    return data

def raycast_volume(volume, shared_slices):
    # Standard raycasting loop over image pixels using volume and shared slices [G]
    return [[0.0]]

def composite_recursive(images, grid_dims):
    # Pairwise alpha compositing of images based on grid hierarchy [G]
    return [[0.0]]
```

## Subtopics

### Boundary Replication for Volume Partitioning
A technique to prevent visual artifacts (black strips) at subvolume boundaries in parallel ray-casting by replicating boundary grid layers between adjacent subvolumes [S2]. It is chosen when partitioning a volume along a plane and the view vector is parallel to the cut plane, which would otherwise cause ray samples in the cut boundary region to be lost [S2]. Unlike general boundary sharing which may involve complex resolution adaptation, this method specifically replicates one layer of the boundary grid into the adjacent subvolume to ensure consistent sampling [S2].

## Operating characteristics
- **Time Complexity:** Rendering time is proportional to the number of voxels plus the cost of sharing and compositing. The resolution adaptation (upsampling/downsampling) adds overhead for adaptive-resolution datasets [S1, unofficial].
- **Memory Complexity:** Increased memory usage due to storing shared slices. In adaptive-resolution cases, the same physical slice may be stored in multiple resolutions for different blocks [S1, unofficial].
- **Strengths:** Eliminates boundary artifacts in block-divided data; handles multi-resolution data effectively; allows independent rendering of subvolumes [S1, unofficial][S2].
- **Limitations:** Requires communication of boundary data between processes; resolution adaptation adds computational cost; compositing must be done recursively [S1, unofficial].
- **When to use:** When using MPI or distributed rendering with 3D block decomposition and/or adaptive-resolution data [S1, unofficial].

## Background
This method was developed in the context of parallel volume visualization of time-varying and adaptive-resolution data. It extends earlier slice-based boundary sharing techniques to 3D blocks [S1, unofficial]. The concept of boundary replication for ray-casting was introduced earlier in the context of parallel ray-traced volume rendering [S2].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] A data distributed, parallel algorithm for ray-traced volume ..., https://www.cs.ucdavis.edu/~ma/papers/PRS93.pdf, fetched 2026-10-27
