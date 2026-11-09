# CSG and Inter-object Intersection — Solid Voxel CSG

Also called: Solid Voxel CSG, VGExclusive, VoxelGrid Boolean Exclusive, VoxelGrid Boolean Operations

## Overview
Solid Voxel CSG performs Constructive Solid Geometry (CSG) operations directly on voxel grids of identical dimensions, specifically computing the exclusive OR (XOR) of multiple input grids to retain only voxels unique to each grid while removing overlaps [S1, unofficial]. It differs from its parent [CSG and Inter-object Intersection](4703-csg-and-inter-object-intersection.md) by operating on pre-voxelized data structures rather than rendering slicemaps with blending [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of $N$ input voxel grids ($G_0, G_1, \dots, G_{N-1}$) [S1, unofficial].
*   **Output:** A single output voxel grid $G_{out}$ of the same dimensions [S1, unofficial].

**Data Structures:**
Each voxel grid $G$ is a 3D boolean volume indexed by integer coordinates $(i, j, k)$ where $0 \le i < W$, $0 \le j < H$, $0 \le k < D$ [S1, unofficial]. Internally, voxels are stored in a flattened 1D array `Grid` and accessed via a linear index calculated from the grid dimensions `SizeUVW` [S1, unofficial].

**Steps:**
1.  Retrieve the list of input voxel grids [S1, unofficial].
2.  If the list is empty, return immediately [S1, unofficial].
3.  Create a clone of the first grid ($G_0$) in the list to serve as the initial output grid $G_{out}$ [S1, unofficial].
4.  Iterate through the remaining grids starting from index $i = 1$ to $N-1$:
    a.  Perform an in-place XOR operation between $G_{out}$ and the current grid $G_i$ [S1, unofficial].
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
        grids: List of VoxelGrid3D objects. Each grid has a 1D array `Grid` 
               and a `SizeUVW` attribute (Point3i) containing dimensions (X, Y, Z).
               All grids must have identical dimensions.
               
    Returns:
        VoxelGrid3D: The resulting XOR grid, or None if input is empty.
    """
    if not grids:
        return None
    
    # Step 3: Clone the first grid to serve as output
    out_grid = grids[0].clone()
    size = out_grid.SizeUVW
    
    # Step 4: Iterate through remaining grids
    for i in range(1, len(grids)):
        current = grids[i]
        
        # Step 4b: Element-wise XOR across all voxels
        for z in range(size.Z):
            for y in range(size.Y):
                for x in range(size.X):
                    # Compute linear index matching the evidence's Grid[voxelIndex] access
                    idx = z * (size.Y * size.X) + y * size.X + x
                    out_grid.Grid[idx] = out_grid.Grid[idx] ^ current.Grid[idx]
                    
    # Step 5: Output is ready
    # Step 6: Add to render list (implementation detail, omitted in pure logic)
    return out_grid
```

**Parameters:**
*   **Constraint:** All input voxel grids must be of the same size (width, height, depth) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Input List:** If the input list is empty, the function returns immediately without producing an output grid [S1, unofficial].
*   **Dimension Mismatch:** If input grids have different dimensions, the operation is undefined or will fail, as the size constraint must be strictly enforced [S1, unofficial].

## Subtopics

### VoxelGrid Boolean Difference
Computes the difference between voxel grids by subtracting subsequent grids from the first [S1, unofficial]. It iterates through the remaining grids starting from index 1 and performs an in-place subtraction operation between the output grid and the current grid [S1, unofficial]. The result is set to the first output parameter and added to the render grid list [S1, unofficial].

### Voxel Grid Boolean Intersection
Computes the logical intersection of multiple 3D voxel grids, retaining only voxels that are active in all input grids [S1, unofficial]. It clones the first grid into an output grid and iterates through the remaining grids starting at index 1, applying a logical AND operation with the output grid for each [S1, unofficial]. An empty input list causes an early return without producing output [S1, unofficial].

### Voxel-Ops and Jack-Ops Boolean Operations
A set of 16 voxel-wise boolean operations performed before writing data into the CFB, governed by Voxel_Ops (for rooms/figurines) and Jack_Ops (for jacks) registers [S2]. The four bits of the index of the boolean function given in either register are exactly the code utilised by the vE to perform the function [S2]. These operations are legal only when the processor is working in Gray_Scale or Binary mode [S2]. In Binary mode, Background colour represents 0 and Foreground colour represents 1 [S2].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance, scaling, strengths, limitations, and usage guidelines: Not given in the sources.

## Background
Defined in the VoxelTools repository [S1, unofficial]. The underlying boolean operation logic is also implemented in hardware processors via Voxel_Ops and Jack_Ops registers [S2].

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
[S2] The voxblt Engine: A Voxel Frame Buffer Processor, https://doi.org/10.2312/eggh/eggh88/085-102, fetched 2026-10-26
