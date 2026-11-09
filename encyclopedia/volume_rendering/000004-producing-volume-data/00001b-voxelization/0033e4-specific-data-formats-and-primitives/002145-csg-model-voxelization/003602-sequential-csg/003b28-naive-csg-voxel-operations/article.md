# Naive CSG Voxel Operations

Also called: NaiveCSG, ProcessingNaive

## Overview
Naive CSG Voxel Operations is a brute-force GPU kernel for performing Constructive Solid Geometry (Boolean) operations (Union, Intersection, Difference) on two voxel grids by processing every voxel independently [S1, unofficial]. It differs from optimized methods by iterating over all voxels without skipping empty regions or using hierarchical structures, ensuring simplicity at the cost of computational efficiency [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Two voxel grids (`grid1`, `grid2`) of identical dimensions and voxel sizes.
*   **Output:** The result of the Boolean operation written into `grid1` (or a third grid, depending on implementation specifics, but the kernel operates on words at coordinates).

**Data Structures:**
The method operates on `VoxelsGrid<T, true>` structures which inherit from `Grid<T>` [S1, unofficial].
*   `mGrid`: A span of type `T` (typically bits or words) storing the voxel data.
*   `mSizeX`, `mSizeY`, `mSizeZ`: Dimensions of the grid.
*   `mVoxelSize`: Physical size of a voxel.
*   `mOriginX`, `mOriginY`, `mOriginZ`: Origin coordinates.
*   Access is performed via a `Bit` struct which utilizes atomic operations (`atomicOr`, `atomicAnd`, `atomicXor`) for device writes [S1, unofficial].

**Parameters:**
*   `blockSize`: Calculated as the next power of 2 of `numWord`, capped at half the device's maximum threads dimension [S1, unofficial].
    *   Formula: `blockSize = NextPow2(numWord, prop.maxThreadsDim[0] / 2)`
*   `gridSize`: Number of blocks, calculated as `(numWord + blockSize - 1) / blockSize` [S1, unofficial].

**Steps:**
1.  Calculate the total number of words to process: `numWord = grid1.Size() * grid1.WordSize()`.
2.  Determine the linear word index from the thread/block indices: `wordIndex = (blockIdx.x * blockDim.x) + threadIdx.x`.
3.  Check bounds: If `wordIndex >= numWord`, terminate the thread.
4.  Compute the linear voxel index: `voxelIndex = wordIndex * grid1.WordSize()`.
5.  Convert the linear voxel index to 3D coordinates $(x, y, z)$ using division and modulo operations on `grid1.VoxelsPerSide()`.
6.  Execute the specific Boolean operation function `Op` on the words at the computed coordinates: `Op(grid1.Word(x, y, z), grid2.Word(x, y, z))`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CUDA Kernel Concept
def naive_csg_kernel(grid1, grid2, Op):
    # 1. Calculate total words
    num_word = grid1.size() * grid1.word_size()
    
    # 2. Calculate linear word index
    word_index = (blockIdx.x * blockDim.x) + threadIdx.x
    
    # 3. Check bounds
    if word_index >= num_word:
        return
    
    # 4. Calculate linear voxel index
    voxel_index = word_index * grid1.word_size()
    
    # 5. Compute 3D coordinates from linear voxel index
    # Assuming grid1.VoxelsPerSide() returns the side length N
    N = grid1.voxels_per_side()
    z = voxel_index // (N * N)
    remainder = voxel_index % (N * N)
    y = remainder // N
    x = remainder % N
    
    # 6. Execute operation
    # Op performs the boolean logic (Union, Intersection, Difference)
    # on the words at (x, y, z) using atomic operations if necessary
    Op(grid1.word(x, y, z), grid2.word(x, y, z))

# Host-side launch configuration
def launch_naive_csg(grid1, grid2, Op):
    # Assert identical dimensions and voxel sizes
    assert grid1.view().voxels_per_side() == grid2.view().voxels_per_side()
    assert grid1.view().voxel_size() == grid2.view().voxel_size()
    
    num_word = grid1.size() * grid1.word_size()
    
    # Calculate block size: next power of 2, capped at half max threads dim
    prop = get_device_properties()
    max_threads = prop.max_threads_dim[0]
    block_size = next_pow2(num_word, max_threads // 2)
    
    # Calculate grid size (number of blocks)
    grid_size = (num_word + block_size - 1) // block_size
    
    # Launch kernel
    naive_csg_kernel<<<grid_size, block_size>>>(grid1, grid2, Op)
```

**Edge Cases and Pitfalls:**
*   **Dimension Mismatch:** The two input grids must have identical dimensions and voxel sizes. This is enforced by assertions at runtime [S1, unofficial].
*   **Precision:** The method relies on integer arithmetic for coordinate mapping; ensure `VoxelsPerSide()` is consistent across grids.

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of voxels in the grid. Every voxel is processed regardless of content.
*   **Memory Complexity:** $O(N)$ for storing the input and output grids.
*   **Performance:** Performance scales linearly with grid size. It is generally slower than optimized methods that skip empty regions or use hierarchical structures (e.g., octrees) [S1, unofficial].
*   **Strengths:** Simple implementation, no complex data structure overhead, predictable memory access patterns for uniform grids.
*   **Limitations:** Inefficient for sparse data or large grids with significant empty space.
*   **When to use:** Small to medium dense grids where implementation simplicity is prioritized over performance.
*   **When not to use:** Large sparse grids or real-time applications requiring high performance on complex CSG operations.

## Background
Naive CSG Voxel Operations is a straightforward approach to voxel-based Boolean operations, implemented as a CUDA kernel in the context of GPU-accelerated mesh voxelization [S1, unofficial]. It serves as a baseline for comparison against more optimized sequential or hierarchical CSG methods.

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
