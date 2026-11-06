# Sequential CSG

Also called: SequentialCSG

## Overview
Sequential CSG is a CPU-based algorithm for performing boolean operations (Union, Intersection, Difference) on two voxel grids by iterating through them word-by-word using word-level parallelism [S1, unofficial]. It differs from GPU-based CSG approaches by operating entirely on the host, processing linear memory blocks of voxels rather than relying on parallel thread execution or image-space blending [S1, unofficial]. It serves as a baseline or fallback implementation for Constructive Solid Geometry (CSG) model voxelization when hardware acceleration is unavailable or when debugging sequential logic is required [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Two voxel grids, `grid1V` and `grid2V`, of identical dimensions and structure.
- Input: A boolean operation type `Op` (Union, Intersection, or Difference).
- Output: A modified `grid1V` containing the result of the boolean operation.

**Data Structures:**
The implementation relies on the `HostVoxelsGrid` structure [S1, unofficial]. Key properties include:
- `Size()`: Total number of voxels in the grid.
- `WordSize()`: Number of voxels packed into a single word (e.g., 64 for `uint64_t`).
- `VoxelsPerSide()`: The side length of the cubic voxel grid (assuming a cubic grid).
- `Word(x, y, z)`: A method to retrieve the word (bitmask) corresponding to the 3D coordinates $(x, y, z)$.

**Parameters:**
- `Op`: A function object or lambda that takes two word values and returns the result of the boolean operation. The source defines specific structs for `Union`, `Intersection`, and `Difference` [S1, unofficial].
- `Types::SEQUENTIAL`: A template parameter that selects this sequential implementation variant [S1, unofficial].

**Steps:**
1. Calculate the total number of words to process:
   $$ \text{numWord} = \lceil \frac{\text{grid1V.Size()}}{\text{grid1V.WordSize()}} \rceil = \frac{\text{grid1V.Size()} + \text{grid1V.WordSize()} - 1}{\text{grid1V.WordSize()}} $$
2. Iterate through each word index $i$ from $0$ to $\text{numWord} - 1$.
3. Calculate the linear voxel index for the start of the current word:
   $$ \text{voxelIndex} = i \times \text{grid1V.WordSize()} $$
4. Derive the 3D coordinates $(z, y, x)$ from the linear voxel index using the grid's `VoxelsPerSide()` dimension $N$:
   $$ z = \frac{\text{voxelIndex}}{N \times N} $$
   $$ y = \frac{\text{voxelIndex} \pmod{N \times N}}{N} $$
   $$ x = \text{voxelIndex} \pmod{N} $$
5. Apply the operation function `Op` to the words at the corresponding $(x, y, z)$ coordinates in `grid1V` and `grid2V`:
   $$ \text{grid1V.Word}(x, y, z) = \text{Op}(\text{grid1V.Word}(x, y, z), \text{grid2V.Word}(x, y, z)) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sequential_csg(grid1, grid2, op_type):
    """
    Perform boolean operation on two voxel grids sequentially.
    
    Args:
        grid1: HostVoxelsGrid instance, modified in place.
        grid2: HostVoxelsGrid instance, read-only.
        op_type: Enum or function defining the boolean operation (Union, Intersection, Difference).
    """
    # Get grid properties
    size = grid1.Size()
    word_size = grid1.WordSize()
    voxels_per_side = grid1.VoxelsPerSide()
    
    # Calculate number of words
    num_word = (size + word_size - 1) // word_size
    
    # Define operation functions based on op_type
    # Note: In the source, these are template structs with __host__ __device__ operators
    # Here we define them as standard Python-like lambdas for clarity
    if op_type == "UNION":
        def apply_op(word1, word2):
            return word1 | word2
    elif op_type == "INTERSECTION":
        def apply_op(word1, word2):
            return word1 & word2
    elif op_type == "DIFFERENCE":
        def apply_op(word1, word2):
            # Difference: A - B = A & ~B
            return word1 & (~word2)
    else:
        raise ValueError("Unknown operation type")

    # Iterate through each word
    for i in range(num_word):
        # Calculate linear voxel index
        voxel_index = i * word_size
        
        # Derive 3D coordinates
        z = voxel_index // (voxels_per_side * voxels_per_side)
        y = (voxel_index % (voxels_per_side * voxels_per_side)) // voxels_per_side
        x = voxel_index % voxels_per_side
        
        # Get words from both grids
        word1 = grid1.Word(x, y, z)
        word2 = grid2.Word(x, y, z)
        
        # Apply operation and store result in grid1
        result = apply_op(word1, word2)
        grid1.SetWord(x, y, z, result)
```

**Edge Cases and Pitfalls:**
- **Grid Dimensions:** The algorithm assumes both grids have identical dimensions and `VoxelsPerSide()` values. Mismatched sizes will lead to out-of-bounds access or incorrect results [S1, unofficial].
- **Word Alignment:** The calculation of `numWord` uses ceiling division to handle grids where the total number of voxels is not a perfect multiple of `WordSize()`. The last word may contain unused bits if the grid size is not aligned [S1, unofficial].
- **Coordinate Derivation:** The derivation of $(z, y, x)$ assumes a specific memory layout (likely row-major or column-major within the 3D grid). The source uses $z$ as the most significant index, $y$ as the middle, and $x$ as the least significant [S1, unofficial].

## Subtopics

### [CSG and Inter-object Intersection](4703-csg-and-inter-object-intersection.md)
Handles intersections between different CSG objects, potentially using different strategies for efficiency.

### [SDF-based Voxelization](6595-sdf-based-voxelization.md)
Uses Signed Distance Fields for voxelization, offering smoother results but higher computational cost.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the total number of voxels. The algorithm iterates through each word once, performing constant-time bitwise operations.
- **Memory Complexity:** $O(1)$ auxiliary space, as the operation is performed in-place on `grid1`.
- **Performance:** As a sequential CPU implementation, it is significantly slower than GPU-based counterparts for large grids. However, it avoids the overhead of data transfer to and from the GPU [S1, unofficial].
- **Strengths:** Simple implementation, no GPU required, easy to debug, low memory overhead.
- **Limitations:** Slow for large grids, limited by CPU clock speed, no parallelism.
- **When to Use:** Small grids, debugging, or environments without GPU support.
- **When Not to Use:** Large-scale volume rendering, real-time applications, or when high performance is required.

## Background
Sequential CSG is part of the `cuda-mesh-voxelization` project by bigmat18, which primarily focuses on GPU-accelerated voxelization but includes a sequential CPU implementation for comparison or fallback purposes [S1, unofficial]. The specific implementation details, including the `HostVoxelsGrid` structure and the boolean operation structs, are found in the `csg.h` header file of the project [S1, unofficial]. The use of `__host__ __device__` qualifiers in the operation structs suggests that the same code might be intended for both CPU and GPU execution, although the sequential variant is explicitly selected via the `Types::SEQUENTIAL` template parameter [S1, unofficial].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
