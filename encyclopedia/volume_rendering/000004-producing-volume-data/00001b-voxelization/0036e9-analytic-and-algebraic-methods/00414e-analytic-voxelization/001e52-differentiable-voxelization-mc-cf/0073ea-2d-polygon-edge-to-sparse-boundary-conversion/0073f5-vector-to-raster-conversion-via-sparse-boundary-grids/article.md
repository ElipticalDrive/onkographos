# Vector-to-Raster Conversion via Sparse Boundary Grids

Also called: 2D polygon-to-raster conversion, 3D block voxelization, Object-based and scan-line based conversion

## Overview
This method converts a set of non-overlapping, closed polyhedra (or polygons) into a fully occupied 3D (or 2D) grid of voxels (or pixels) where every voxel holds a specific value [S1]. It solves the problem of generic, high-performance voxelization by decoupling the process into two distinct stages: an object-based boundary extraction and a scan-line-based interior filling [S1]. Unlike intersection-based methods that compute distances for every voxel or object-type-specific algorithms requiring different logic for lines, surfaces, and parametric curves, this approach uses bitwise XOR operations to handle any closed watertight primitive (polyhedra, ellipsoids, splines) with uniform complexity [S1]. It builds on [2D Polygon Edge to Sparse Boundary Conversion](22873-2d-polygon-edge-to-sparse-boundary-conversion.md) by providing the mechanism to fill the interior defined by those boundaries.

## How it works
The algorithm operates in two stages. Stage 1 converts vector boundaries into a "sparse boundary grid." Stage 2 converts this sparse grid into a "full voxel grid" using bitwise XOR prefix operations [S1].

### Inputs and Outputs
**Input:**
*   A set of non-overlapping, closed polyhedra (3D) or polygons (2D).
*   Each object has an associated numerical value (ID, class label, semantic value) [S1].
*   Objects may be nested or touching [S1].
*   If the objects do not fill the entire space, the remainder is assigned a default background value, typically $0$ [S1].
*   Topology tolerance: It is sufficient that each boundary "knows" the two objects on either side. Boundaries between entities may appear twice in the dataset (once for each side, separating the object from "background") [S1].

**Output:**
*   A 3D grid of voxels (or 2D grid of pixels) where every voxel has a value [S1].

### Data Structures
**Sparse Boundary Grid:**
*   A grid representation where boundaries are not completely closed.
*   Non-zero values indicate transitions between objects or between object and background.
*   The non-zero values do *not* form a connected path around each area [S1].
*   Layout: Same dimensions as the target output grid.

**Full Voxel Grid:**
*   A grid where every voxel has a value.
*   Layout: Same dimensions as the target output grid.

### Mathematical Definitions
The method relies on the Bitwise Exclusive OR (XOR) operator, denoted as $\oplus$ or `xor`.

**Properties of XOR [S1]:**
1.  **Commutativity:** $a \oplus b = b \oplus a$. This allows handling two entities sharing a common boundary independently.
2.  **Associativity:** $(a \oplus b) \oplus c = a \oplus (b \oplus c)$. This allows grouping operations.
3.  **Identity and Self-Inverse:** $a \oplus a = 0$ and $a \oplus 0 = a$.
    *   Consequence: $(a \oplus c) \oplus (b \oplus c) = a \oplus b$. This property cancels out common boundaries between entities [S1].

**XOR Infix:**
Applied to a sequence $A = a_0, a_1, \dots, a_n$, it produces a sequence of length $n$ by applying XOR to all pairs of neighboring elements:
$$ \text{Infix}(A) = (a_0 \oplus a_1, \, a_1 \oplus a_2, \, \dots, \, a_{n-1} \oplus a_n) $$

**XOR Prefix:**
Applied to a sequence $A = a_0, a_1, \dots, a_n$, it produces a sequence of length $n+1$ by applying XOR to all contiguous subsequences starting from $a_0$:
$$ \text{Prefix}(A) = (a_0, \, a_0 \oplus a_1, \, a_0 \oplus a_1 \oplus a_2, \, \dots, \, a_0 \oplus \dots \oplus a_n) $$

### Algorithm Steps

#### Stage 1: Create Sparse Boundary Grid
This stage is object-based. It converts the vector input into the sparse boundary grid.
1.  Initialize a grid of the target resolution with the default background value (typically $0$) [S1].
2.  For each input object (polyhedron/polygon):
    *   Rasterize its boundary voxels/pixels.
    *   Assign the object's value to these boundary voxels.
    *   *Note:* If a boundary is shared by two objects, it may be processed twice (once for each object), or the topology must support identifying both neighbors. The method is tolerant of boundaries appearing twice, separating the object from "background" [S1].
3.  The result is a grid where non-zero values mark boundaries. This is the "Sparse Boundary Grid" [S1].

*Implementation Note:* The source describes Stage 1 as creating the sparse grid from vector input. It does not provide a specific algorithm for *how* the vector boundary is mapped to the grid indices other than stating it is "object-based" and "boundary rasterization" [S1]. The source explicitly states that Stage 1 can accommodate ellipsoids, splines, etc., as long as they are watertight [S1].

#### Stage 2: Create Full Voxel Grid (Scan-line Based)
This stage fills the interior using XOR prefix operations. It can be applied row-wise or column-wise in 2D, and plane-wise along the depth axis in 3D [S1].

1.  Let $G$ be the Sparse Boundary Grid.
2.  For each row (or column, or depth slice) $R$ in $G$:
    a.  Let $R = [v_0, v_1, \dots, v_{n-1}]$ be the sequence of values in the row.
    b.  Compute the XOR Prefix of $R$:
        $$ P = \text{Prefix}(R) $$
        $$ P[i] = \bigoplus_{k=0}^{i} v_k $$
    c.  Replace the row in $G$ with $P$.
3.  The result is the Full Voxel Grid [S1].

**Restoration (Inverse Operation):**
To convert a Full Voxel Grid back to a Sparse Boundary Grid, apply XOR Infix to each row/column/slice [S1].
1.  Let $G$ be the Full Voxel Grid.
2.  For each row $R = [v_0, v_1, \dots, v_{n-1}]$:
    a.  Compute the XOR Infix of $R$:
        $$ S = \text{Infix}(R) $$
        $$ S[i] = v_i \oplus v_{i+1} \quad \text{for } i = 0 \dots n-2 $$
    b.  Prepend the first element $v_0$ to $S$ to maintain total pixel count:
        $$ S_{final} = [v_0] + S $$
    c.  Replace the row in $G$ with $S_{final}$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def xor_prefix(sequence):
    """
    Computes the XOR prefix of a sequence.
    Input: list of integers [a0, a1, ..., an]
    Output: list of integers [a0, a0^a1, a0^a1^a2, ..., a0^...^an]
    """
    result = []
    current_xor = 0
    for value in sequence:
        current_xor = current_xor ^ value
        result.append(current_xor)
    return result

def xor_infix(sequence):
    """
    Computes the XOR infix of a sequence, prepending the first element.
    Input: list of integers [a0, a1, ..., an]
    Output: list of integers [a0, a0^a1, a1^a2, ..., an-1^an]
    """
    if not sequence:
        return []
    
    # Prepend the first element
    result = [sequence[0]]
    
    # Compute infix for the rest
    for i in range(len(sequence) - 1):
        val = sequence[i] ^ sequence[i+1]
        result.append(val)
        
    return result

def stage_2_fill_sparse_to_full(sparse_grid, dimensions):
    """
    Stage 2: Convert Sparse Boundary Grid to Full Voxel Grid.
    sparse_grid: 3D list or array of shape (D, H, W) or 2D (H, W).
    dimensions: tuple (D, H, W) or (H, W).
    """
    full_grid = [[0 for _ in range(dimensions[-1])] for _ in range(dimensions[-2])]
    if len(dimensions) == 3:
        full_grid = [[[0 for _ in range(dimensions[2])] for _ in range(dimensions[1])] for _ in range(dimensions[0])]

    # Apply XOR prefix along the last axis (rows in 2D, rows in 3D slices)
    # Note: The source mentions applying to rows or columns. 
    # Standard implementation typically processes one axis at a time.
    # Here we process along the last dimension (width).
    
    if len(dimensions) == 2:
        H, W = dimensions
        for h in range(H):
            row = sparse_grid[h]
            filled_row = xor_prefix(row)
            full_grid[h] = filled_row
    else:
        D, H, W = dimensions
        for d in range(D):
            for h in range(H):
                row = sparse_grid[d][h]
                filled_row = xor_prefix(row)
                full_grid[d][h] = filled_row
                
    return full_grid

def stage_1_extract_boundaries(vector_objects, grid_shape, default_value=0):
    """
    Stage 1: Convert vector objects to Sparse Boundary Grid.
    This is a placeholder for the specific boundary rasterization logic,
    as the source does not provide the exact algorithm for mapping 
    vector primitives to grid indices, only that it is 'object-based'.
    
    vector_objects: List of objects, each with a 'value' and a 'boundary' representation.
    grid_shape: (D, H, W) or (H, W).
    default_value: Background value (0).
    """
    # Initialize grid with default value
    if len(grid_shape) == 2:
        sparse_grid = [[default_value for _ in range(grid_shape[1])] for _ in range(grid_shape[0])]
    else:
        sparse_grid = [[[default_value for _ in range(grid_shape[2])] for _ in range(grid_shape[1])] for _ in range(grid_shape[0])]

    # For each object, rasterize its boundary
    for obj in vector_objects:
        obj_value = obj.value
        # Get boundary voxels/pixels for this object
        # The source states: "each boundary 'knows' the two polygons... 
        # acceptable when every boundary separates the object from 'background'"
        # This implies we simply write the object's value to its boundary voxels.
        boundary_voxels = get_boundary_voxels(obj, grid_shape) # [G] Helper to determine boundary indices
        
        for voxel in boundary_voxels:
            sparse_grid[voxel] = obj_value
            
    return sparse_grid

def get_boundary_voxels(obj, grid_shape):
    """
    [G] Placeholder for the specific geometric rasterization of the object's boundary.
    The source does not detail this step, only that it is object-based and handles 
    polyhedra, ellipsoids, etc.
    """
    # Implementation depends on the specific primitive type (polyhedron, ellipsoid, etc.)
    # This function should return a list of grid coordinates (indices) 
    # that constitute the boundary of the object.
    return []
```

### Parameters
*   **Input Topology:** Tolerant. Boundaries may appear twice (once for each side, separating object from background) [S1].
*   **Default Value:** Typically $0$ for background [S1].
*   **Value Range:** The source illustrates 8-bit values (0-255), but the principle applies to 16, 32, or 64-bit values [S1].

### Edge Cases and Pitfalls
*   **Precision:** The method relies on bitwise XOR, which is exact for integer values. No floating-point precision issues are introduced in Stage 2 [S1].
*   **Boundaries:** The method assumes watertight objects. Non-watertight objects may result in incorrect filling if the XOR prefix does not return to the background value at the end of the scan line [S1].
*   **Overlapping Objects:** The input must be non-overlapping. Overlapping objects would violate the assumption that boundaries separate distinct entities or background [S1].

## Operating characteristics
*   **Time Complexity:** Stage 1 depends on the complexity of the boundary rasterization (object-based). Stage 2 is linear with respect to the number of voxels/pixels, $O(N)$, where $N$ is the total number of voxels, because it performs a single pass of XOR operations per row/column/slice [S1].
*   **Memory Complexity:** $O(N)$ for storing the grid. The sparse boundary grid and full voxel grid have the same dimensions [S1].
*   **Performance:** The use of bitwise XOR operations allows for execution at CPU highest speed [S1]. This is significantly faster than intersection-based methods that compute distances and intersections for every voxel [S1].
*   **Strengths:**
    *   Generic: Works for any closed polyhedron/polygon, including ellipsoids, splines, and NURBS, as long as they are watertight [S1].
    *   Tolerant of input topology: Does not require explicit connectivity between objects; boundaries can be treated independently [S1].
    *   Fast: Leverages CPU bitwise operations [S1].
*   **Limitations:**
    *   Requires watertight objects.
    *   Stage 1 (boundary extraction) may be complex for non-polygonal primitives if not handled by a specialized rasterizer.
    *   The method is described as a "novel approach" in the source, but the source also notes that "best" algorithms have not crystallized out and GPU-based solutions exist (e.g., binvox) which may offer different trade-offs [S1].

## Background
*   **Origin:** The method was presented by Nourian et al. in 2016 [S1].
*   **History:**
    *   Early studies by Kaufman & Shimony (1986), Kaufman (1987), and Wang and Kaufman (1993) developed algorithms for voxelizing 3D lines, polygons, and parametric curves, but these were object-type specific [S1].
    *   Cohen-or & Kaufman (1997) revisited connectivity issues.
    *   Huang et al. (1998) presented algorithms for planes and surfaces, but these involved high-complexity spatial operations [S1].
    *   Laine (2013) presented a topological method, considered one of the best for its generality, with an implementation by Nourian et al. (2016) [S1].
    *   The specific "Sparse Boundary Grid" approach is a generic solution that avoids object-type specificity and intersection-based complexity [S1].
*   **Usage:**
    *   Used in ILWIS (Gorte et al. 1988) for 2D GIS [S1].
    *   Related libraries include binvox (Min 2016) and Voxelization toolkit (Milossramek 2013) [S1].
    *   Standard libraries like GDAL provide software for this purpose [S1].

## Sources
[S1] RASTERIZATION AND VOXELIZATION OF TWO- AND THREE-DIMENSIONAL SPACE PARTITIONINGS, https://doi.org/10.5194/isprs-archives-xli-b4-283-2016, fetched 2026-10-26
