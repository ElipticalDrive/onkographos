# Reduced Volume Texture with Box Average

Also called: Box average reduction, Coarse grid representation, Multi-resolution volume textures, Reduced slice store

## Overview
Reduced Volume Texture with Box Average is a technique for handling volumetric datasets that exceed the maximum texture dimensions supported by the GPU (e.g., 2048 voxels per axis) [S1, unofficial]. It solves the problem of rendering failure and main-thread stalling by maintaining multiple representations of the data: a full-resolution grid for high-quality rendering and a coarser grid created by box averaging for preview or low-resolution display [S1, S2, unofficial]. This approach allows the viewport to select the representation that fits the device's texture limits, ensuring continuous rendering even when the full volume cannot be loaded into GPU memory [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of source frames (slices) delivered by a loader, potentially arriving asynchronously.
*   **Output:** A reduced volume texture (coarse grid) stored in CPU memory, which is uploaded to the GPU as needed. The size of this reduced copy is bounded by the maximum texture size supported by the hardware (e.g., $256 \times 256 \times 256$ int16, which is 32 MB) [S2, unofficial].

**Data Structures:**
*   **VoxelGrid:** A type representing the grid descriptor and the box average arithmetic [S1, unofficial].
*   **CompositeVoxelManager:** An object that holds the different representations (full and reduced) and reads the best one for the current viewport [S1, unofficial].
*   **Quality Record:** Tracks what the loader has delivered and at which quality level [S1, unofficial].
*   **Reduced Texture:** A 3D texture array or volume texture where each slice corresponds to a reduced representation of the source data.

**Algorithm Steps:**
1.  **Box Averaging Reduction:** Reduce a grid by a box on each axis to compute the coarser representation [S1, unofficial]. For a target voxel at $(i, j, k)$ in the reduced grid, the value is the average of the source voxels within the corresponding box.
2.  **Batch Derivation:** The volume reduces each delivered frame into a reduced representation in CPU memory in batches, outside the render loop [S2, unofficial]. This is managed by `CompositeVoxelManager.createVoxelRepresentation`, `refreshDerivedFrames`, and `deriveInBatches` with `scheduleIdle` [S2, unofficial].
3.  **Handling Missing Frames:** If a frame has not arrived, it must not be treated as zero, which would incorrectly lower the average [S1, unofficial]. A reduced slice reads only the frames that have arrived [S1, unofficial]. A replicate (previous value) counts only when no frame of the box arrived [S1, unofficial].
4.  **Rounding and Storage:** To prevent integer truncation, the reduction uses `fillPlaneByAverage`, which reads the target column from a table, keeps one row of sums, and applies rounding before storing in an integer texture [S1, unofficial].
5.  **Incremental Fill:** A render fills at most 16 reduced slices that hold data. The render stops after 8 ms, but only after filling one slice [S1, unofficial]. The texture requests another render, and the next fill starts where the last fill stopped [S1, unofficial]. This is controlled by the `rendering.reducedTextureFill` configuration object [S1, unofficial].
6.  **Fast Path Optimization:** The derived path uses the `fillPlane` fast path of the statistic. The `getAtIJK` reducer is identified as slow for $i$ or $j$ factors and should be avoided in the derived path [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def box_average_reduce(source_grid, reduction_factor):
    """
    Reduces a source grid by averaging blocks of voxels.
    source_grid: 3D array of shape (D, H, W)
    reduction_factor: integer, the factor by which to reduce each axis
    """
    src_d, src_h, src_w = source_grid.shape
    dst_d = src_d // reduction_factor
    dst_h = src_h // reduction_factor
    dst_w = src_w // reduction_factor
    
    # Initialize sums and counts
    sums = np.zeros((dst_d, dst_h, dst_w), dtype=np.float64)
    counts = np.zeros((dst_d, dst_h, dst_w), dtype=np.int32)
    
    for d in range(dst_d):
        for h in range(dst_h):
            for w in range(dst_w):
                # Define the box in source coordinates
                d_start = d * reduction_factor
                h_start = h * reduction_factor
                w_start = w * reduction_factor
                
                # Sum valid voxels in the box
                # Note: In a real implementation, 'valid' means frames that have arrived
                # For this simplified pseudocode, we assume all source voxels are valid
                block = source_grid[d_start:d_start+reduction_factor, 
                                    h_start:h_start+reduction_factor, 
                                    w_start:w_start+reduction_factor]
                
                # Count non-zero or valid entries if necessary
                # [G] Assuming all entries are valid for the average calculation
                valid_count = block.size 
                sums[d, h, w] = np.sum(block)
                counts[d, h, w] = valid_count
                
    # Compute average with rounding
    # Avoid division by zero if a box has no valid data
    avg = np.where(counts > 0, sums / counts, 0.0)
    
    # Round to nearest integer for storage in integer texture
    reduced_grid = np.round(avg).astype(np.int16)
    return reduced_grid

def fill_reduced_slice_incrementally(reduced_texture, target_slice_index, max_slices_per_render=16, max_time_ms=8):
    """
    Fills reduced slices incrementally to avoid stalling the main thread.
    """
    # [G] Implementation details for pacing and state management are omitted
    # as they depend on the specific rendering engine's scheduler.
    pass
```

**Parameters:**
*   `rendering.reducedTextureFill`: Configuration object that sets the limits for reduced texture fill work per render [S1, unofficial]. Specifically, it limits the number of slices filled per render (e.g., 16) and the time spent (e.g., 8 ms) [S1, unofficial].
*   **Maximum Texture Size:** The size of the reduced copy is at most the size of the texture supported by the device (e.g., 2048 voxels/axis limit, or a reduced size like $256 \times 256 \times 256$) [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Missing Frames:** If a frame has not arrived, treating it as zero causes the average to be too low [S1, unofficial]. The fix is to read only the frames that arrived and count replicates only when no frame of the box arrived [S1, unofficial].
*   **Integer Truncation:** Integer textures previously truncated the average value [S1, unofficial]. The fix is to use `fillPlaneByAverage` which applies rounding [S1, unofficial].
*   **Main Thread Stall:** Deriving the full volume previously stalled the main thread for seconds [S2, unofficial]. This is avoided by deriving for each delivery in batches [S2, unofficial].
*   **Slow Reducer:** The `getAtIJK` reducer is slow for $i$ or $j$ factors [S2, unofficial]. The `fillPlane` fast path should be used instead [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The reduction is performed in batches on the CPU, outside the render loop, to avoid stalling the main thread [S2, unofficial]. The fill of the reduced texture is incremental, limited to 16 slices or 8 ms per render [S1, unofficial].
*   **Memory Complexity:** The reduced texture size is bounded by the maximum texture size supported by the GPU (e.g., 32 MB for a $256 \times 256 \times 256$ int16 texture) [S2, unofficial].
*   **Performance:** This method allows rendering of volumes that exceed GPU texture limits by providing a lower-quality preview [S1, S2, unofficial]. It prevents rendering stalls and failures [S1, S2, unofficial].
*   **Strengths:** Enables rendering of large volumes on devices with limited texture memory; prevents main-thread blocking; provides a progressive display experience [S1, S2, unofficial].
*   **Limitations:** The reduced texture is a lower-quality approximation; requires CPU-based reduction; incremental filling may result in partial updates if the render is interrupted [S1, S2, unofficial].
*   **When to Use:** When the volume size exceeds the GPU's maximum texture dimensions or when a quick preview is needed before full loading [S1, S2, unofficial].
*   **When Not to Use:** When high-fidelity rendering of the full resolution is required and the device supports it [S1, unofficial].

## Background
This technique was developed as part of the cornerstone3D project, specifically in PR #2931 and Issue #2966 [S1, S2, unofficial]. It addresses the limitation of 2048 voxels per axis on known devices, which previously caused texture allocation failures and empty viewers [S1, unofficial]. The work is based on task T2 of PR #2931 and includes refactoring of the `IVoxelManager` interface, addition of `VoxelGrid` and box average arithmetic, and the creation of `CompositeVoxelManager` [S1, unofficial]. The approach was refined to avoid main-thread stalls by moving reduction to CPU batches and implementing incremental texture filling [S2, unofficial].

## Sources
[S1] feat(core): multi-resolution volume textures, the base for task T2, https://github.com/cornerstonejs/cornerstone3D/pull/2931, fetched 2026-10-26 (unofficial)
[S2] Volume textures: brick copies in the render, a recorded update copy, paced renders, and a preload that states its loader, https://github.com/cornerstonejs/cornerstone3D/issues/2966, fetched 2026-10-27 (unofficial)
