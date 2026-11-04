# Slice Filling with BFS Propagation

Also called: FillXY2, FillXZ2, FillYZ2

## Overview
Slice Filling with BFS Propagation extends [Slice Filling (Simple)](1691-slice-filling-simple.md) by propagating filled voxels to adjacent neighbors within the slice plane using a Breadth-First Search (BFS) [S1, unofficial]. It differs from its parent by actively searching for and filling empty neighbors that are not solid in either the main voxel grid or the current buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid (`voxels_`), a 2D buffer for the current slice (`voxels_buffer_`), and the slice bounding box coordinates ($lx, ux, ly, uy, lz, uz$) [S1, unofficial].
*   **Output:** The `voxels_buffer_` updated with all voxels filled by the BFS propagation [S1, unofficial].

**Data Structures:**
*   `voxels_`: The main 3D voxel grid storing the state of each voxel (solid or empty) [S1, unofficial].
*   `voxels_buffer_`: A 2D buffer corresponding to the current slice being processed [S1, unofficial].
*   `BfsSolid`: A helper function that performs the BFS propagation from a given voxel index [S1, unofficial].

**Steps:**
The algorithm scans the slice plane along two axes, performing a forward pass and a backward pass [S1, unofficial]. In the forward scan, it iterates $y$ from $ly$ to $uy$ and $z$ from $lz$ to $uz$ [S1, unofficial]. For each position, it computes the voxel index and loads the state from `voxels_` [S1, unofficial]. If the voxel is solid, it breaks the forward scan [S1, unofficial]. If the voxel is not solid, it checks its 4 neighbors in the slice plane using predefined offsets [S1, unofficial]. For each neighbor within bounds, it computes the neighbor's index and loads the combined state from `voxels_` and `voxels_buffer_` [S1, unofficial]. If the neighbor is not solid, it calls `BfsSolid` on the neighbor's index [S1, unofficial]. In the backward scan, it iterates $z$ from $uz$ down to $lz$ [S1, unofficial]. It computes the voxel index and loads the state [S1, unofficial]. If solid, it breaks the backward scan [S1, unofficial]. If not solid, it checks its 4 neighbors in the slice plane [S1, unofficial]. It loads the combined state and calls `BfsSolid` if the neighbor is not solid [S1, unofficial]. The logic for $FillXZ2(y)$ and $FillXY2(z)$ follows the same pattern, adjusting the iteration axes and neighbor offsets accordingly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Predefined neighbor offsets for the YZ plane
DI_4_YZ = [(0, 1), (0, -1), (1, 0), (-1, 0)]

def compute_index(x, y, z, stride_y, stride_z):
    """Converts 3D coordinates to a linear buffer index [G]."""
    return x * stride_y * stride_z + y * stride_z + z

def FillYZ2(x, ly, uy, lz, uz, voxels_, voxels_buffer_, BfsSolid, stride_y, stride_z):
    # Forward scan: Iterate y from ly to uy
    for y in range(ly, uy + 1):
        # Inner loop: Iterate z from lz to uz
        for z in range(lz, uz + 1):
            voxel_index = compute_index(x, y, z, stride_y, stride_z)
            is_solid = voxels_[voxel_index]
            
            if is_solid:
                break
                
            for dy, dz in DI_4_YZ:
                ny, nz = y + dy, z + dz
                if ly <= ny <= uy and lz <= nz <= uz:
                    neighbor_index = compute_index(x, ny, nz, stride_y, stride_z)
                    if not voxels_[neighbor_index] and not voxels_buffer_[neighbor_index]:
                        BfsSolid(neighbor_index, voxels_, voxels_buffer_)

    # Backward scan: Iterate z from uz down to lz
    # Note: sources specify iterating z from uz down to lz; inner y loop added for complete 2D plane coverage [G]
    for z in range(uz, lz - 1, -1):
        for y in range(uy, ly - 1, -1):
            voxel_index = compute_index(x, y, z, stride_y, stride_z)
            is_solid = voxels_[voxel_index]
            
            if is_solid:
                break
                
            for dy, dz in DI_4_YZ:
                ny, nz = y + dy, z + dz
                if ly <= ny <= uy and lz <= nz <= uz:
                    neighbor_index = compute_index(x, ny, nz, stride_y, stride_z)
                    if not voxels_[neighbor_index] and not voxels_buffer_[neighbor_index]:
                        BfsSolid(neighbor_index, voxels_, voxels_buffer_)
```

**Parameters:**
*   `BfsSolid`: A function that performs BFS propagation from a voxel index [S1, unofficial]. It typically adds the voxel to a queue, marks it as filled, and processes its neighbors [G].

**Edge Cases and Pitfalls:**
*   **Bounds Checking:** Neighbors must be validated against the slice boundaries ($ly, uy, lz, uz$) to prevent out-of-bounds memory access [S1, unofficial].
*   **Solid Voxels:** The scan terminates immediately when a solid voxel is encountered in the main grid, preventing propagation through solid geometry [S1, unofficial].
*   **Combined State:** Propagation only occurs if a neighbor is empty in *both* the main grid and the buffer; solidity in either state halts propagation for that neighbor [S1, unofficial].

## Subtopics

### [BFS Light Propagation](2164-bfs-light-propagation.md)
BFS Light Propagation is a voxel-based lighting algorithm that propagates light levels from emitters across a padded voxel chunk using a Breadth-First Search (BFS). It solves the problem of efficiently illuminating non-opaque blocks in a voxel grid while attenuating light intensity with distance. It differs from its parent, [Slice Filling with BFS Propagation](1909-slice-filling-with-bfs-propagation.md), by operating on the full 3D padded chunk rather than just the slice plane, and by explicitly modeling light attenuation per step.

## Operating characteristics
Not given in the sources.

## Background
The method belongs to a voxelization pipeline, specifically within the "Ray-Cast Style Voxel Filling" stage [S1, unofficial]. The implementation originates from the unofficial GitHub repository `topskychen/voxelizer` [S1, unofficial]. The repository contains components for ray casting, collision detection, and thread pooling, indicating its use in converting mesh data into voxel representations [S1, unofficial].

## Sources
[S1] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
