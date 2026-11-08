# Mathematical Morphology on Voxelization
Also called: Separable Morphological Operations on Voxels, Voxel Erosion/Dilation

## Overview
Mathematical Morphology on Voxelization performs erosion and dilation on solid voxel grids to create hierarchical structures for rapid neighborhood queries, path finding, or visibility checks [S1, unofficial]. It differs from standard morphological dilation by explicitly leveraging the separability of logical operations on binary voxel data to achieve high performance through iterative, axis-aligned passes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A solid voxelization grid (binary volume where foreground voxels are 1 and background are 0).
- Output: An eroded or dilated voxel grid of the same dimensions.

**Data Structures:**
- The voxel grid is stored as a 3D array of bits or integers. To optimize memory and bandwidth, voxels are packed into 32-bit integers along the Z-axis (depth), such that one integer represents a column of 32 voxels [S1, unofficial].
- Let $V(x, y, z)$ be the voxel value at coordinates $(x, y, z)$.
- Let $I(x, y)$ be the 32-bit integer representing the column at $(x, y)$, where the $k$-th bit corresponds to $V(x, y, k)$ for $k \in [0, 31]$.

**Steps:**
1. **Separable Decomposition:** Decompose the 3D morphological operation into two stages:
   - Stage 1: Perform 2D erosion/dilation along the X and Y axes for each Z-slice (or column).
   - Stage 2: Perform 1D erosion/dilation along the Z axis (within the packed integers).
2. **2D Pass (X/Y):** For each voxel $(x, y, z)$, apply the logical kernel in the XY plane.
   - **Dilation:** $V_{out}(x, y, z) = V_{in}(x, y, z) \lor V_{in}(x-1, y, z) \lor V_{in}(x+1, y, z) \lor V_{in}(x, y-1, z) \lor V_{in}(x, y+1, z)$ (for a 3x3 cross or square kernel).
   - **Erosion:** $V_{out}(x, y, z) = V_{in}(x, y, z) \land V_{in}(x-1, y, z) \land V_{in}(x+1, y, z) \land V_{in}(x, y-1, z) \land V_{in}(x, y+1, z)$.
   - This step can be implemented using texture lookups or direct memory access if the grid is in memory.
3. **1D Pass (Z):** Apply the operation along the Z-axis using bit manipulation on the packed integers $I(x, y)$.
   - **Dilation (Z):** Shift the integer $I(x, y)$ by $\pm 1$ bit and perform a bitwise OR.
     $$ I_{out}(x, y) = I_{in}(x, y) \lor (I_{in}(x, y) \ll 1) \lor (I_{in}(x, y) \gg 1) $$
     *Note:* Care must be taken with boundary conditions. Bits shifted out of the 32-bit integer must be retrieved from adjacent integers in the Z-direction if the grid is larger than 32 units in Z, or handled via boundary checks [S1, unofficial].
   - **Erosion (Z):** Shift and perform bitwise AND.
     $$ I_{out}(x, y) = I_{in}(x, y) \land (I_{in}(x, y) \ll 1) \land (I_{in}(x, y) \gg 1) $$
4. **Iteration:** To achieve larger kernel sizes (e.g., radius $r$), repeat the separable passes $r$ times. Each iteration expands the effective neighborhood by 1 unit in each direction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def morphological_voxel_operation(grid, operation='dilate', radius=1):
    """
    grid: 3D list/array of integers (packed Z-columns) or bits.
    operation: 'dilate' or 'erode'
    radius: number of iterations
    """
    current_grid = grid
    for _ in range(radius):
        # Stage 1: 2D Pass (X/Y)
        # Assuming grid is accessed as grid[x][y][z_packed_int]
        # For simplicity, assume we process bit by bit or use vectorized ops
        next_grid_2d = process_2d_pass(current_grid, operation)
        
        # Stage 2: 1D Pass (Z) using bit shifts
        next_grid = []
        for x in range(width):
            col_x = []
            for y in range(height):
                packed = next_grid_2d[x][y]
                if operation == 'dilate':
                    # Dilate along Z: OR with shifted versions
                    # Handle boundaries: assume 0 padding or clamp
                    shifted_up = packed << 1
                    shifted_down = packed >> 1
                    # Mask to keep only 32 bits if necessary
                    packed_out = packed | shifted_up | shifted_down
                else: # erode
                    shifted_up = packed << 1
                    shifted_down = packed >> 1
                    packed_out = packed & shifted_up & shifted_down
                col_x.append(packed_out)
            next_grid.append(col_x)
        current_grid = next_grid
    return current_grid

def process_2d_pass(grid, operation):
    """
    Process 2D neighborhood for each Z-slice.
    Returns a new grid with 2D morphological results.
    """
    # Implementation depends on specific 2D kernel.
    # For a 3x3 square kernel:
    # Dilate: OR with neighbors in X and Y.
    # Erode: AND with neighbors in X and Y.
    # This step is separable into X then Y passes.
    pass # Placeholder for specific 2D logic as per source [S1, unofficial]
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** When shifting bits in the Z-direction, bits shifted out of the 32-bit integer must be handled. If the grid height exceeds 32, adjacent integers in the Z-dimension must be accessed to retrieve the missing bits [S1, unofficial].
- **Precision:** The method relies on binary logic, so precision issues are minimal, but boundary handling is critical to avoid artifacts.
- **Degenerate Input:** Empty or fully filled grids result in empty or fully filled outputs, respectively.

## Subtopics

### [Voxel Erosion / Fracture](10370-voxel-erosion-fracture.md)
Voxel Erosion / Fracture is a GPU-accelerated iterative process that removes voxels from a solid voxelized model to simulate physical fracture or weathering effects. It differs from standard mathematical morphology by introducing probabilistic removal based on noise buffers and configurable thresholds, allowing for irregular, natural-looking fragmentation rather than uniform shrinking. It builds on [Mathematical Morphology on Voxelization](4702-mathematical-morphology-on-voxelization.md) by replacing deterministic logical operations with stochastic compute shader passes.

## Operating characteristics
- **Time Complexity:** $O(N \cdot r)$, where $N$ is the number of voxels and $r$ is the radius (number of iterations). Each iteration is $O(N)$ due to the separable nature.
- **Memory Complexity:** $O(N)$ for storing the voxel grid.
- **Performance:** The method is faster than lengthy preprocesses because it is purely image-based and relies on the solid voxelization as the only step involving actual scene geometry [S1, unofficial].
- **Strengths:** High performance on GPU hardware, especially with integer texture support. Simple implementation using bitwise operations.
- **Limitations:** Limited to binary or quantized data. Boundary handling can be complex for large Z-dimensions.
- **When to Use:** Path finding, visibility checks, and hierarchical neighborhood queries on voxelized data.
- **When Not to Use:** When non-binary data or complex kernel shapes are required.

## Background
The method originates from single-pass GPU solid voxelization techniques, where morphological operations are applied directly to the voxelized representation of geometry [S1, unofficial]. It leverages the separability of logical operations to optimize performance on GPU hardware, particularly those supporting 32-bit integer textures [S1, unofficial].

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
