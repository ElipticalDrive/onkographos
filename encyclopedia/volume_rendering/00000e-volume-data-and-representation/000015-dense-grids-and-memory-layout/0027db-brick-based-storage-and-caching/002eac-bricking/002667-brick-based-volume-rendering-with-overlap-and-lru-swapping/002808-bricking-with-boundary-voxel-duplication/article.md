# Bricking with Boundary Voxel Duplication

Also called: Boundary layer correction, Clipping with pre-integrated volume rendering, Texture Bricking, Volume Boundary Replication

## Overview
Bricking with Boundary Voxel Duplication is a technique to solve incorrect interpolation at texture tile boundaries in 3D volume rendering by duplicating a plane of voxels at each brick boundary [S1, S2, unofficial]. It prevents black strips at cut boundaries in composited images by ensuring features at the cut are not dropped [S2, unofficial]. It sets itself apart from sibling methods like [Level-wise Texture Merging](5274-level-wise-texture-merging.md) by focusing on spatial continuity of the scalar field at fixed brick boundaries rather than hierarchical level-of-detail transitions, and from [Wang Cubes for Volume Rendering](6980-wang-cubes-for-volume-rendering.md) by addressing hardware texture interpolation artifacts rather than geometric surface representation.

## How it works
**Inputs and Outputs:**
- Input: A 3D volume grid divided into subvolumes (bricks).
- Output: A composited image where ray samples across brick boundaries are consistent, avoiding artifacts.

**Data Structures:**
- **Brick Structure:** Neighboring bricks share a common plane of voxels at their boundary to enable consistent trilinear interpolation [S1, unofficial].
- **Boundary Replication:** One layer of the boundary grid is replicated at each subvolume [S2, unofficial].

**Steps:**
1. Divide the volume grid into subvolumes.
2. Identify the boundary planes between adjacent subvolumes.
3. For each subvolume, replicate the voxel layer at the boundary into the adjacent subvolume's data structure. For example, if Subvolume 1 includes layer 1 to layer $k$, and Subvolume 2 includes layer $k$ to layer $n$, layer $k$ is replicated in Subvolume 2 [S2, unofficial].
4. Load each brick into GPU texture memory.
5. Perform ray casting. When a ray crosses a brick boundary, the texture unit uses the duplicated voxels to perform correct trilinear interpolation across the boundary [S1, unofficial].

**Formulas:**
- Let $V$ be the volume grid.
- Let $B_i$ be a brick.
- Let $P$ be a boundary plane between $B_i$ and $B_j$.
- The voxel value $v$ at position $x$ on $P$ is stored in both $B_i$ and $B_j$.
- Trilinear interpolation at boundary: $I(x) = \sum_{k=0}^{7} w_k v_k$, where $v_k$ are the 8 neighboring voxels. If $x$ is on the boundary, the duplicated voxels ensure $v_k$ are consistent across $B_i$ and $B_j$ [S1, unofficial].

**Parameters:**
- Replication depth: 1 layer of voxels [S2, unofficial].

**Edge Cases and Pitfalls:**
- If the view vector is parallel to the cut plane and boundary replication is not used, a black strip will appear at each cut boundary in the composited image [S2, unofficial].
- At the boundary between texture tiles, interpolation is incorrect because the texture unit does not have enough information to consistently interpolate across texture boundaries [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def brick_with_boundary_voxel_duplication(volume_grid, brick_size):
    """
    Divides volume_grid into bricks and duplicates boundary voxels.
    volume_grid: 3D array of voxel values.
    brick_size: tuple (dx, dy, dz) defining brick dimensions.
    """
    bricks = {}
    # Determine brick grid dimensions
    nx, ny, nz = volume_grid.shape
    bx, by, bz = brick_size
    
    # Iterate over brick coordinates
    for i in range(0, nx, bx):
        for j in range(0, ny, by):
            for k in range(0, nz, bz):
                # Define brick bounds
                x_start, x_end = i, min(i + bx, nx)
                y_start, y_end = j, min(j + by, ny)
                z_start, z_end = k, min(k + bz, nz)
                
                # Extract brick data
                brick_data = volume_grid[x_start:x_end, y_start:y_end, z_start:z_end]
                
                # Duplicate boundary voxels
                # Duplicate at x_end if not at volume boundary
                if x_end < nx:
                    # Duplicate the plane at x_end-1 into x_end
                    brick_data[:, :, -1] = volume_grid[x_end-1, y_start:y_end, z_start:z_end]
                
                # Duplicate at y_end if not at volume boundary
                if y_end < ny:
                    # Duplicate the plane at y_end-1 into y_end
                    brick_data[:, -1, :] = volume_grid[x_start:x_end, y_end-1, z_start:z_end]
                
                # Duplicate at z_end if not at volume boundary
                if z_end < nz:
                    # Duplicate the plane at z_end-1 into z_end
                    brick_data[-1, :, :] = volume_grid[x_start:x_end, y_start:y_end, z_end-1]
                
                bricks[(i, j, k)] = brick_data
    
    return bricks
```

## Subtopics

### [Continuous Level Transitions](5275-continuous-level-transitions.md)
Maintains continuity of the 3D scalar field at boundaries where adjacent bricks are rendered at different levels of detail (LOD). It adapts the textures of finer-level bricks to interpolate correctly from the coarser level, restricting transitions to a difference of at most one level.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Increases memory usage due to duplicated boundary voxels. The exact overhead depends on the number of bricks and their size.
- **Measured Performance:** For large volume data sets, the bricking strategy is extremely inefficient, resulting in significantly lower frame rates compared to 2D-textures [S1, unofficial].
- **Strengths:** Superior image quality compared to 2D-texture based solutions; removes significant drawbacks of 2D approaches; hardware support for trilinear interpolation provides a natural means of increasing the sampling rate to obtain images of high quality [S1, unofficial].
- **Limitations:** Extremely inefficient for large volume data sets; significantly lower frame rates for large data compared to 2D-textures [S1, unofficial].
- **When to use:** When high image quality is required and the volume data set is not extremely large.
- **When not to use:** For large volume data sets where frame rate is critical.

## Background
The technique was introduced in the context of volume rendering techniques for general purpose graphics hardware [S1, unofficial]. It was also discussed in the context of a data distributed parallel algorithm for ray-traced volume rendering [S2, unofficial]. The 3D-texture based approach has proven superior in terms of image quality, removing some of the significant drawbacks while preserving almost all the benefits [S1, unofficial].

## Sources
[S1] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA270501: A Data Distributed Parallel Algorithm for Ray-Traced Volume Rendering, https://archive.org/details/DTIC_ADA270501, fetched 2026-10-26 (unofficial)
