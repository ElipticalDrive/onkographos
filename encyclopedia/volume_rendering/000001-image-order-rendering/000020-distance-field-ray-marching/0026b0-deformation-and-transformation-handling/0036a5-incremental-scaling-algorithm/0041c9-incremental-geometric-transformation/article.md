# Incremental Geometric Transformation

Also called: Incremental Transformation

## Overview
Incremental Geometric Transformation is a method for applying affine transformations (translation, scaling, rotation) to voxel primitives by computing the new position of each voxel based on the already-transformed position of its immediate predecessor, using a maximum of three additions per coordinate instead of full matrix multiplication [S1][S2, unofficial]. It sets itself apart from direct matrix multiplication by exploiting spatial coherence between neighboring voxels, significantly reducing arithmetic operations at the cost of requiring all volume elements (including empty ones) to be processed, making it particularly suitable for parallel architectures [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid defined by source coordinates $(x, y, z)$, and an affine transformation matrix $M$ (typically $4 \times 4$ for homogeneous coordinates) or specific transformation parameters (translation vector $\vec{T}$, scaling factors, rotation angles).
- Output: A transformed voxel grid where each voxel's position is updated according to $M$.

**Data Structures:**
- Source Grid: A 3D array of voxels. The exact layout (e.g., row-major, column-major) depends on the hardware implementation, but the algorithm assumes a regular grid structure where neighbors are accessible via simple index increments [S1].
- Transformation Matrix: A $4 \times 4$ matrix $M$ representing the affine transform.
- Intermediate Position: A 3D (or 4D homogeneous) vector storing the transformed position of the current voxel.

**Numbered Steps:**
1. **Initial Transformation:** Compute the transformed position of the first voxel in the grid (typically the origin or the starting corner of the primitive) using a standard matrix-vector multiplication: $\vec{p}_{0}' = M \cdot \vec{p}_{0}$ [S2, unofficial].
2. **Incremental Update:** For each subsequent voxel in the grid, compute its transformed position by adding the incremental change derived from the transformation of its immediate predecessor. This exploits the fact that the relative displacement between neighboring voxels in the source space is constant and can be precomputed or derived from the transformation matrix [S1][S2, unofficial].
3. **Overlap Resolution:** If the source and destination primitives overlap, perform the transformation in a predefined sequence determined by an orientation algorithm to prevent source voxels from being overwritten before they are translated [S1]. This is necessary when no scratch memory is available to hold old voxel information [S1].

**Formulas:**
Let $\vec{p}_{i}$ be the source position of voxel $i$, and $\vec{p}_{i}'$ be its transformed position.
For the first voxel ($i=0$):
$$ \vec{p}_{0}' = M \cdot \vec{p}_{0} $$
For subsequent voxels ($i > 0$), let $\Delta \vec{p}_{i}$ be the incremental change. The algorithm states that the position of the next voxel is determined by its predecessor's new position plus a fixed increment derived from the transformation [S1][S2, unofficial].
Specifically, for a translation by vector $\vec{T} = (T_x, T_y, T_z)$, the incremental update is simply:
$$ \vec{p}_{i}' = \vec{p}_{i-1}' + \vec{T} $$
For general affine transformations, the incremental change for each coordinate requires three additions per coordinate, implying the precomputation of the transformed basis vectors or deltas [S1][S2, unofficial]. The text notes "three additions per coordinate" [S2, unofficial], which suggests that for a general affine transform, the algorithm might compute the transformed unit vectors and add them incrementally, or use a table lookup method where "nine table lookup operations and nine additions" are replaced by incremental steps [S2, unofficial]. However, the core claim is "maximum of 3 additions/increments" per voxel [S1]. This likely refers to the three components of the position vector $(x, y, z)$, each updated by one addition of a precomputed delta [S1].

**Parameters:**
- Transformation Matrix $M$: Defined by the specific geometric operation (translation, rotation, scaling).
- Grid Dimensions: $N_x, N_y, N_z$.

**Edge Cases and Pitfalls:**
- **Overlap:** Overlapping source and destination primitives can cause data corruption if voxels are overwritten before being read. This is resolved by processing voxels in a specific order based on an orientation algorithm [S1].
- **Empty Voxels:** The method requires transforming all volume elements, including empty ones, which increases computational load if not parallelized [S2, unofficial].
- **Precision:** The incremental nature may accumulate floating-point errors over large grids, though the sources do not specify mitigation strategies [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_geometric_transformation(voxel_grid, transformation_matrix):
    """
    Applies an affine transformation to a voxel grid using incremental updates.
    
    Args:
        voxel_grid: 3D array of voxels with source coordinates.
        transformation_matrix: 4x4 affine transformation matrix.
        
    Returns:
        Transformed voxel grid.
    """
    # Get grid dimensions
    Nx, Ny, Nz = voxel_grid.shape
    
    # Initialize output grid
    transformed_grid = empty_grid_of_same_shape(voxel_grid)
    
    # Step 1: Transform the first voxel (origin) using matrix multiplication
    # Assuming voxel_grid[0,0,0] has a position attribute or we compute it from indices
    p0_src = voxel_grid[0, 0, 0].position # Homogeneous coordinate (x, y, z, 1)
    p0_dst = matrix_vector_multiply(transformation_matrix, p0_src)
    
    # Store the transformed position for the first voxel
    transformed_grid[0, 0, 0].position = p0_dst
    
    # Precompute incremental deltas for each axis based on the transformation matrix
    # For a general affine transform, the delta for moving +1 in x, y, or z in source space
    # is the corresponding column of the transformation matrix (in homogeneous coords).
    # However, the source mentions "3 additions per coordinate", implying a simpler update.
    # We assume the deltas are precomputed or derived from the matrix.
    # Let's assume we have precomputed deltas dx, dy, dz for each axis step.
    # In a pure translation T, dx = T, dy = T, dz = T.
    # For rotation/scaling, the deltas are the transformed unit vectors.
    
    # Note: The exact derivation of deltas is not fully specified in the sources.
    # We assume a helper function exists to get the incremental vector for a step in a given axis.
    delta_x = get_incremental_vector(transformation_matrix, axis='x')
    delta_y = get_incremental_vector(transformation_matrix, axis='y')
    delta_z = get_incremental_vector(transformation_matrix, axis='z')
    
    # Step 2: Incrementally update the remaining voxels
    # Iterate through the grid in a predefined sequence to handle overlaps if necessary
    # The sources mention an "orientation algorithm" for overlap resolution [S1].
    # We assume a simple raster order for now, but in practice, the order matters.
    
    for z in range(Nz):
        for y in range(Ny):
            for x in range(1, Nx): # Start from 1 to use previous x
                # Get previous voxel's transformed position
                prev_pos = transformed_grid[x-1, y, z].position
                
                # Incrementally compute current position
                # Add the delta for moving one step in x
                curr_pos = vector_add(prev_pos, delta_x)
                
                transformed_grid[x, y, z].position = curr_pos
            
            for x in range(Nx):
                for y in range(1, Ny): # Start from 1 to use previous y
                    prev_pos = transformed_grid[x, y-1, z].position
                    curr_pos = vector_add(prev_pos, delta_y)
                    transformed_grid[x, y, z].position = curr_pos
                
                for y in range(Ny):
                    for x in range(Nx):
                        for z in range(1, Nz): # Start from 1 to use previous z
                            prev_pos = transformed_grid[x, y, z-1].position
                            curr_pos = vector_add(prev_pos, delta_z)
                            transformed_grid[x, y, z].position = curr_pos

    return transformed_grid

def get_incremental_vector(matrix, axis):
    """
    Computes the incremental vector for a unit step in the given axis.
    This is essentially the transformed unit vector for that axis.
    """
    if axis == 'x':
        unit_vec = [1, 0, 0, 0]
    elif axis == 'y':
        unit_vec = [0, 1, 0, 0]
    elif axis == 'z':
        unit_vec = [0, 0, 1, 0]
    
    # Transform the unit vector
    transformed = matrix_vector_multiply(matrix, unit_vec)
    
    # Return the 3D part of the result (ignoring w for position delta)
    return [transformed[0], transformed[1], transformed[2]]

def matrix_vector_multiply(matrix, vector):
    """Standard 4x4 matrix by 4-vector multiplication."""
    result = [0, 0, 0, 0]
    for i in range(4):
        sum_val = 0
        for j in range(4):
            sum_val += matrix[i][j] * vector[j]
        result[i] = sum_val
    return result

def vector_add(v1, v2):
    """Component-wise addition of two 3D vectors."""
    return [v1[0] + v2[0], v1[1] + v2[1], v1[2] + v2[2]]
```

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the total number of voxels, with a very low constant factor (3 additions per voxel) compared to $O(N)$ with a higher constant for matrix multiplication [S1][S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing the voxel grid. No additional significant memory is required beyond the transformation matrix and intermediate position vectors [S1].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of voxels. Performance is highly dependent on parallelization, as all voxels (including empty ones) must be processed [S2, unofficial].
- **Strengths:** Reduces arithmetic operations significantly (3 additions vs. matrix multiplication); exploits spatial coherence; suitable for parallel architectures [S1][S2, unofficial].
- **Limitations:** Requires processing all voxels, including empty ones; potential for overlap issues if not handled with a specific ordering algorithm; accumulation of floating-point errors over large grids [S1][S2, unofficial].
- **When to use:** When hardware supports parallel processing and arithmetic operations are expensive; when the transformation is affine and the grid is dense or parallelization is feasible [S2, unofficial].
- **When not to use:** When the grid is sparse and most voxels are empty (wastes computation); when overlap resolution is complex and cannot be handled by a simple ordering algorithm [S1][S2, unofficial].

## Background
The Incremental Geometric Transformation technique was developed as part of the Voxblt Engine, a voxel frame buffer processor, in the late 1980s [S1]. It was designed to perform geometric operations (translation, scaling, rotation) efficiently by exploiting inter- and intra-scan-line coherency [S1]. The method was implemented in hardware, utilizing parallel memory and barrel shifters to accelerate the incremental updates [S1]. It is also discussed in the context of volume visualization and rendering techniques, where it is noted for its suitability to parallel architectures [S2, unofficial].

## Sources
[S1] The voxblt Engine: A Voxel Frame Buffer Processor, https://doi.org/10.2312/eggh/eggh88/085-102, fetched 2026-10-26
[S2] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
