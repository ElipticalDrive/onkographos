# Voxel Occupancy Boundary Detection

Also called: Voxel Boundary Detection, Occupancy Grid Boundary Extraction

## Overview
Voxel Occupancy Boundary Detection identifies voxels in a 3D occupancy grid that lie on the interface between occupied and unoccupied space [S1, unofficial]. It solves the problem of converting a discrete boolean volume into a set of boundary indicators required for surface extraction algorithms, such as Boolean Voxel Meshing [G]. It differs from [Octree Traversal for Surface Extraction](2113-octree-traversal-for-surface-extraction.md) by operating on a fixed-resolution grid rather than a hierarchical tree, and from [VoxelGrid Mesh Generation](2714-voxelgrid-mesh-generation.md) by outputting a boolean mask of boundary voxels rather than a mesh directly [G].

## How it works
**Inputs:** A 3D boolean occupancy grid `occ` of shape $(D, H, W)$, where `True` indicates an occupied voxel and `False` indicates an unoccupied voxel [S1, unofficial].
**Outputs:** A 3D boolean boundary mask `boundary` of shape $(D-1, H-1, W-1)$, where `True` indicates a voxel is on the boundary [S1, unofficial].

**Data Structures:**
*   `occ`: 3D array of booleans representing the input occupancy grid.
*   `occupied`: 3D array of booleans, representing voxels fully enclosed by occupied neighbors.
*   `unoccupied`: 3D array of booleans, representing voxels fully enclosed by unoccupied neighbors.
*   `boundary`: 3D array of booleans, the result.

**Steps:**
1.  **Compute Fully Occupied Voxels:** A voxel is considered "fully occupied" (interior) if all 8 of its corner neighbors in the occupancy grid are `True` [S1, unofficial]. This is computed using logical AND across all 8 shifted slices of the grid [S1, unofficial].
2.  **Compute Fully Unoccupied Voxels:** A voxel is considered "fully unoccupied" (exterior) if all 8 of its corner neighbors are `False` [S1, unofficial]. This is computed using logical OR on the negated neighbors, then negating the result [S1, unofficial].
3.  **Compute Boundary:** A voxel is on the boundary if it is neither fully occupied nor fully unoccupied [S1, unofficial].

**Formulas:**
Let $O_{i,j,k}$ be the value of the voxel at index $(i,j,k)$.
The fully occupied mask $I_{i,j,k}$ is:
$$ I_{i,j,k} = \bigwedge_{d \in \{0,1\}, h \in \{0,1\}, w \in \{0,1\}} O_{i+d, j+h, k+w} $$
The fully unoccupied mask $E_{i,j,k}$ is:
$$ E_{i,j,k} = \neg \left( \bigvee_{d \in \{0,1\}, h \in \{0,1\}, w \in \{0,1\}} \neg O_{i+d, j+h, k+w} \right) $$
The boundary mask $B_{i,j,k}$ is:
$$ B_{i,j,k} = \neg I_{i,j,k} \land \neg E_{i,j,k} $$
where indices $i, j, k$ range from $0$ to $D-2$, $0$ to $H-2$, and $0$ to $W-2$ respectively [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_occupancy_boundary_detection(occ):
    """
    Detects boundary voxels in a 3D boolean occupancy grid.
    
    Args:
        occ: 3D list of booleans of shape (D, H, W).
        
    Returns:
        boundary: 3D list of booleans of shape (D-1, H-1, W-1).
    """
    D, H, W = len(occ), len(occ[0]), len(occ[0][0])
    
    # Initialize output arrays for the interior grid
    occupied = [[False] * (W - 1) for _ in range(H - 1)]
    unoccupied = [[False] * (W - 1) for _ in range(H - 1)]
    boundary = [[False] * (W - 1) for _ in range(H - 1)]
    
    for i in range(D - 1):
        for j in range(H - 1):
            for k in range(W - 1):
                # Extract the 8 corner neighbors of the 2x2x2 block
                corners = [
                    occ[i][j][k], occ[i][j][k+1],
                    occ[i][j+1][k], occ[i][j+1][k+1],
                    occ[i+1][j][k], occ[i+1][j][k+1],
                    occ[i+1][j+1][k], occ[i+1][j+1][k+1]
                ]
                
                # Step 1: Fully occupied if all 8 corners are True
                if all(corners):
                    occupied[i][j][k] = True
                # Step 2: Fully unoccupied if all 8 corners are False
                elif not any(corners):
                    unoccupied[i][j][k] = True
                # Step 3: Boundary if neither condition holds
                else:
                    boundary[i][j][k] = True
                    
    return boundary
```

**Parameters:**
*   None.

**Edge Cases:**
*   **Grid Boundaries:** The slicing operations reduce the grid size by 1 in each dimension [S1, unofficial]. Voxels on the outer boundary of the input grid are excluded from the output masks because they lack a complete 2x2x2 neighborhood [S1, unofficial]. [G] Standard practice pads the input with `False` (unoccupied) to preserve the original shape, but the source computes the mask strictly on the interior grid.
*   **Precision:** Boolean logic is exact. No floating-point issues occur.

## Subtopics

### Binary Volume Cell Classification
Binary Volume Cell Classification is a variant used in octree-based surface extraction [S2, unofficial]. It classifies octree leaf cells in a binary volume to determine if an isosurface intersects the cell [S2, unofficial]. It differs from the standard voxel boundary detection by operating on octree cells and explicitly identifying "heterogeneous" cells and "bipolar edges" [S2, unofficial]. It is chosen when hierarchical representation is required [G].
*   **Definition:** Classification of octree leaf cells in a binary volume (values 0 for internal, 1 for external) to determine if an isosurface intersects the cell [S2, unofficial].
*   **Steps:**
    1.  Check the 8 corner voxels of the cell [S2, unofficial].
    2.  If all corner voxels have the same value, classify as homogeneous (non-heterogeneous) [S2, unofficial].
    3.  If at least one corner voxel has a different value than the others, classify as heterogeneous [S2, unofficial].
    4.  For heterogeneous cells, identify bipolar edges: edges where the endpoint voxels have different values (0 and 1) [S2, unofficial].
*   **Pitfalls:** Bipolar edges indicate that the isosurface intersects the edge at least once. This is the primary indicator for surface extraction within the cell [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(D \times H \times W)$, where $D, H, W$ are the dimensions of the grid [G]. The operation involves a constant number of shifts and logical operations per voxel [G].
*   **Memory Complexity:** $O(D \times H \times W)$ for the input and output masks [G]. Intermediate masks (`occupied`, `unoccupied`) also require $O(D \times H \times W)$ memory [G].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Simple, parallelizable, and efficient for dense grids [G].
*   **Limitations:** Does not handle hierarchical structures. Boundary conditions at the grid edges are ambiguous without padding [G].
*   **When to use:** When a fixed-resolution occupancy grid is available and a quick boundary mask is needed for meshing [G].
*   **When not to use:** When the volume is sparse or hierarchical (use Octree Traversal) [G].

## Background
The method is used to extract surfaces from neural occupancy fields and differentiable volumetric rendering pipelines [S1, unofficial]. The concept of checking corner neighbors to determine cell heterogeneity is also found in binary volume surface extraction literature [S2, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
