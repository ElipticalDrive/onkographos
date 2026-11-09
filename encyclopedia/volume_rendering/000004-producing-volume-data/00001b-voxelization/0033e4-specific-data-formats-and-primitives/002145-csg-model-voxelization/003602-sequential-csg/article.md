# Sequential CSG

Also called: SequentialCSG
## Overview
Sequential CSG is a CPU-based algorithm for performing boolean operations (Union, Intersection, Difference) on two voxel grids using word-level parallelism [S1, unofficial]. It differs from GPU-based or hierarchical approaches by iterating linearly through the grid data structure, applying bitwise operations to chunks of voxels (words) rather than individual voxels, ensuring simplicity and deterministic execution at the cost of parallel throughput [S1, unofficial]. It serves as the sequential baseline for CSG Model Voxelization [2497-csg-model-voxelization.md].

## How it works
**Inputs and Outputs:**
- Input: Two voxel grids (`grid1V`, `grid2V`) of identical dimensions and structure.
- Input: A boolean operation type (`Op`), such as Union, Intersection, or Difference.
- Output: A modified `grid1V` containing the result of the operation.

**Data Structures:**
The method operates on `HostVoxelsGrid` structures [S1, unofficial].
- `VoxelsPerSide()`: The integer dimension $N$ of the cubic grid.
- `WordSize()`: The number of voxels packed into a single word (e.g., 64 for `uint64_t`).
- `Size()`: The total number of voxels ($N^3$).
- `Word(x, y, z)`: A function returning a reference to the word (integer type) containing the voxel at coordinates $(x, y, z)$.

**Steps:**
1. Calculate the total number of words to process:
   $$ \text{numWord} = \lceil \frac{\text{grid1V.Size()}}{\text{grid1V.WordSize()}} \rceil = \frac{\text{grid1V.Size()} + \text{grid1V.WordSize()} - 1}{\text{grid1V.WordSize()}} $$
2. Iterate through each word index $i$ from $0$ to $\text{numWord} - 1$.
3. Calculate the linear voxel index corresponding to the start of the current word:
   $$ \text{voxelIndex} = i \times \text{grid1V.WordSize()} $$
4. Derive the 3D coordinates $(z, y, x)$ from the linear index using the grid's side length $S = \text{grid1V.VoxelsPerSide()}$:
   $$ z = \left\lfloor \frac{\text{voxelIndex}}{S \times S} \right\rfloor $$
   $$ y = \left\lfloor \frac{\text{voxelIndex} \pmod{S \times S}}{S} \right\rfloor $$
   $$ x = \text{voxelIndex} \pmod S $$
5. Retrieve the word references for both grids at $(x, y, z)$.
6. Apply the boolean operation to the words. For example, for Union, perform a bitwise OR; for Intersection, a bitwise AND.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sequential_csg(grid1, grid2, op_type):
    """
    Perform boolean CSG operation on two voxel grids sequentially.
    
    Args:
        grid1: HostVoxelsGrid object, modified in place.
        grid2: HostVoxelsGrid object, read-only.
        op_type: Enum indicating the operation (UNION, INTERSECTION, DIFFERENCE).
    """
    # 1. Calculate number of words
    total_voxels = grid1.Size()
    word_size = grid1.WordSize()
    num_word = (total_voxels + word_size - 1) // word_size
    
    # 2. Iterate through words
    for i in range(num_word):
        # 3. Calculate linear voxel index for the start of the word
        voxel_index = i * word_size
        
        # 4. Derive 3D coordinates
        voxels_per_side = grid1.VoxelsPerSide()
        z = voxel_index // (voxels_per_side * voxels_per_side)
        y = (voxel_index % (voxels_per_side * voxels_per_side)) // voxels_per_side
        x = voxel_index % voxels_per_side
        
        # 5. Get word references
        word1 = grid1.Word(x, y, z)
        word2 = grid2.Word(x, y, z)
        
        # 6. Apply operation
        if op_type == Op.UNION:
            word1 |= word2
        elif op_type == Op.INTERSECTION:
            word1 &= word2
        elif op_type == Op.DIFFERENCE:
            word1 &= ~word2
        elif op_type == Op.VOID:
            word1 = 0
```

**Parameters:**
- `Op`: A function object or lambda that defines the boolean logic. Typical implementations use bitwise operators `|`, `&`, `~` [S1, unofficial].
- `Types::SEQUENTIAL`: A template parameter selecting this implementation variant [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Grid Mismatch:** The algorithm assumes `grid1` and `grid2` have identical `Size()` and `VoxelsPerSide()`. Mismatched dimensions will cause out-of-bounds access or incorrect results [S1, unofficial].
- **Word Boundary:** The calculation of `numWord` uses ceiling division to handle grid sizes that are not exact multiples of `WordSize()`. The last word may contain unused bits (padding) which must be ignored or masked if strict voxel precision is required [G].
- **Precision:** The method relies on the underlying integer type's width (e.g., 64 bits). Ensure the word type matches the hardware's efficient word size for optimal performance [S1, unofficial].

## Subtopics

### [Naive CSG Voxel Operations](8926-naive-csg-voxel-operations.md)
A brute-force GPU kernel that processes every voxel independently. It differs from Sequential CSG by lacking word-level parallelism and hierarchical skipping, making it simpler but computationally less efficient for large grids.

## Operating characteristics
- **Time Complexity:** $O(N^3 / W)$, where $N$ is the grid side length and $W$ is the word size (bits per word). The algorithm processes each word once [S1, unofficial].
- **Memory Complexity:** $O(N^3)$ for storing the voxel grids. The algorithm is in-place for `grid1` [S1, unofficial].
- **Performance:** As a sequential CPU implementation, it lacks the parallel throughput of GPU-based CSG methods. It is suitable for small to medium grids or as a baseline for correctness verification [S1, unofficial].
- **Strengths:** Simple implementation, deterministic, no synchronization overhead, low memory footprint.
- **Limitations:** Slow for large grids due to single-threaded execution; does not skip empty regions [S1, unofficial].
- **When to use:** When implementing a reference version of CSG voxelization or when grid sizes are small enough that CPU performance is acceptable [S1, unofficial].
- **When not to use:** For large-scale volume rendering or real-time applications requiring high throughput [S1, unofficial].

## Background
Sequential CSG was introduced as part of the `cuda-mesh-voxelization` project by bigmat18 [S1, unofficial]. It provides a CPU-based alternative to GPU-accelerated voxel boolean operations, serving as a foundational component in the CSG Model Voxelization pipeline [2497-csg-model-voxelization.md]. The method leverages word-level parallelism to improve efficiency over per-voxel operations while maintaining the simplicity of a sequential algorithm [S1, unofficial].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
