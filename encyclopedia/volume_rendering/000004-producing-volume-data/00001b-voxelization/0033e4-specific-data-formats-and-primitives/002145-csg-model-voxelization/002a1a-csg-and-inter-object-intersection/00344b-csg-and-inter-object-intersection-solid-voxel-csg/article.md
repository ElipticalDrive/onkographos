# CSG and Inter-object Intersection — Solid Voxel CSG

Also called: Solid Voxel CSG, VGExclusive, VoxelGrid Boolean Exclusive, VoxelGrid Boolean Operations

## Overview
Solid Voxel CSG performs Constructive Solid Geometry (CSG) operations directly on voxel grids of identical dimensions, specifically computing the exclusive OR (XOR) of multiple input grids to retain only voxels unique to each grid while removing overlaps [S1, unofficial]. It differs from its parent [CSG and Inter-object Intersection](4703-csg-and-inter-object-intersection.md) by operating on pre-voxelized data structures rather than rendering slicemaps with blending, allowing for direct bitwise manipulation of the voxel volume [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of $N$ input voxel grids ($G_0, G_1, \dots, G_{N-1}$). All grids must have the same dimensions $(W, H, D)$ and voxel size [S1, unofficial].
*   **Output:** A single output voxel grid $G_{out}$ of the same dimensions [S1, unofficial].

**Data Structures:**
Each voxel grid $G$ is a 3D array of boolean values (or bits) indexed by integer coordinates $(i, j, k)$ where $0 \le i < W$, $0 \le j < H$, $0 \le k < D$ [S1, unofficial]. The grid may be stored as a flattened 1D array or a 3D array; the logical operation is element-wise [S1, unofficial].

**Steps:**
1.  Retrieve the list of input voxel grids.
2.  If the list is empty, return immediately (no output) [S1, unofficial].
3.  Create a clone of the first grid ($G_0$) in the list to serve as the initial output grid $G_{out}$ [S1, unofficial].
4.  Iterate through the remaining grids starting from index $i = 1$ to $N-1$:
    a.  Perform an in-place XOR operation between $G_{out}$ and the current grid $G_i$.
    b.  For each voxel index $v$ in the grid:
        $$ G_{out}[v] \leftarrow G_{out}[v] \oplus G_i[v] $$
        where $\oplus$ is the bitwise exclusive OR operation [S1, unofficial].
5.  Set the resulting $G_{out}$ to the first output parameter [S1, unofficial].
6.  Add $G_{out}$ to the render grid list for visualization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solid_voxel_csg_xor(grids):
    """
    Perform Solid Voxel CSG XOR operation on a list of voxel grids.
    
    Args:
        grids: List of VoxelGrid3D objects. Each grid has dimensions (W, H, D)
               and supports indexing grids[i][x][y][z] or grids[i].get_value(x,y,z).
               All grids must have identical dimensions.
               
    Returns:
        VoxelGrid3D: The resulting XOR grid, or None if input is empty.
    """
    if not grids:
        return None
    
    # Step 3: Clone the first grid to serve as output
    # Assuming VoxelGrid3D has a clone() method and supports element-wise access
    out_grid = grids[0].clone()
    
    # Step 4: Iterate through remaining grids
    for i in range(1, len(grids)):
        current_grid = grids[i]
        
        # Ensure dimensions match (constraint check)
        if out_grid.dimensions != current_grid.dimensions:
            raise ValueError("All voxel grids must have the same size.")
            
        # Perform in-place XOR
        # Iterate over all voxels
        for x in range(out_grid.width):
            for y in range(out_grid.height):
                for z in range(out_grid.depth):
                    # Get boolean values (1 for active, 0 for inactive)
                    val_out = out_grid.get_value(x, y, z)
                    val_curr = current_grid.get_value(x, y, z)
                    
                    # XOR operation
                    out_grid.set_value(x, y, z, val_out ^ val_curr)
                    
    # Step 5: Output is ready
    # Step 6: Add to render list (implementation detail, omitted in pure logic)
    return out_grid
```

**Parameters:**
*   **Constraint:** All input voxel grids must be of the same size (width, height, depth) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Input List:** If the input list is empty, the function returns immediately without producing an output grid [S1, unofficial].
*   **Dimension Mismatch:** If input grids have different dimensions, the operation is undefined or will fail. The implementation must enforce identical dimensions [S1, unofficial].

## Subtopics

### VoxelGrid Boolean Difference
Computes the difference between voxel grids by subtracting subsequent grids from the first. It iterates through the remaining grids starting from index 1 and performs an in-place subtraction (difference) operation between the output grid and the current grid [S1, unofficial]. The result is set to the first output parameter and added to the render grid list [S1, unofficial].

### Voxel Grid Boolean Intersection
Computes the logical intersection of multiple 3D voxel grids, retaining only voxels that are active in all input grids [S1, unofficial]. It clones the first grid into an output grid and iterates through the remaining grids starting at index 1, applying a logical AND operation with the output grid for each [S1, unofficial]. An empty input list causes an early return without producing output [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot W \cdot H \cdot D)$, where $N$ is the number of input grids and $W, H, D$ are the grid dimensions. Each voxel is visited once per grid operation [S1, unofficial].
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the output grid, plus $O(N \cdot W \cdot H \cdot D)$ for storing input grids if they are not streamed [S1, unofficial].
*   **Strengths:** Direct manipulation of voxel data allows for fast boolean operations without the overhead of rasterization or blending [S1, unofficial].
*   **Limitations:** Requires all input grids to be of the same size, which may necessitate padding or resampling if source objects have different bounding boxes [S1, unofficial].

## Background
Solid Voxel CSG is part of the VoxelTools library, an unofficial toolset for voxel manipulation [S1, unofficial]. It provides basic boolean operations (XOR, Difference, Intersection) on 3D voxel grids, enabling constructive solid geometry at the voxel level [S1, unofficial].

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
