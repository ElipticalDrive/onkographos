# Incremental Geometric Transformation

Also called: Incremental Transformation

## Overview
Incremental Geometric Transformation is a method for applying affine transformations (translation, scaling, rotation) to voxel primitives by computing the new position of each voxel based on the already-transformed position of its immediate predecessor, using a maximum of three additions per voxel instead of full matrix multiplication [S1][S2, unofficial]. It sets itself apart from direct matrix multiplication by exploiting spatial coherence between neighboring voxels, significantly reducing arithmetic operations at the cost of requiring all volume elements (including empty ones) to be processed, making it particularly suitable for parallel architectures [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid defined by source coordinates $(x, y, z)$, and an affine transformation matrix $M$ [S1][S2, unofficial].
- Output: A transformed voxel grid where each voxel's position is updated according to $M$ [S1].

**Data Structures:**
- Source Grid: A regular 3D array where neighbors are accessible via simple index increments [S1].
- Transformation Matrix: A $4 \times 4$ matrix $M$ representing the affine transform [S2, unofficial].
- Intermediate Position: A 4D homogeneous vector storing the transformed position of the current voxel [S2, unofficial].

**Numbered Steps:**
1. **Initial Transformation:** Compute the transformed position of the first grid point using one matrix-vector multiplication [S2, unofficial].
2. **Incremental Update:** Compute the transformed position of subsequent voxels by adding precomputed incremental deltas derived from the transformation matrix to the transformed position of their immediate predecessor, exploiting inter- and intra-scan-line coherency [S1][S2, unofficial].
3. **Overlap Resolution:** If source and destination primitives overlap, perform the transformation in a predefined sequence determined by an orientation algorithm to prevent source voxels from being overwritten before they are translated [S1]. This is necessary because no scratch memory is available to hold old voxel information [S1].

**Formulas:**
Let $\vec{p}_{i,j,k}$ be the source position of voxel $(i,j,k)$, and $\vec{p}_{i,j,k}'$ be its transformed position. For the first voxel $(0,0,0)$:
$$ \vec{p}_{0,0,0}' = M \cdot \vec{p}_{0,0,0} $$
For subsequent voxels, the algorithm uses fixed increments corresponding to the transformed basis vectors:
$$ \Delta_x = M \cdot [1,0,0,0]^T, \quad \Delta_y = M \cdot [0,1,0,0]^T, \quad \Delta_z = M \cdot [0,0,1,0]^T $$
The position updates as $\vec{p}_{i,j,k}' = \vec{p}_{i-1,j,k}' + \Delta_x$ (and similarly for $y$ and $z$) [S1]. The sources state the method requires "three additions/increments" per voxel [S1], though one source notes "three additions per coordinate" [S2, unofficial]; this refers to updating the three spatial components using the precomputed deltas.

**Parameters:**
- Transformation Matrix $M$: Defined by the specific geometric operation (translation, rotation, scaling) [S1][S2, unofficial].
- Grid Dimensions: $N_x, N_y, N_z$ [S1].

**Edge Cases and Pitfalls:**
- **Overlap:** Overlapping source and destination primitives can cause data corruption if voxels are overwritten before being read, resolved by a predefined processing order [S1].
- **Empty Voxels:** The method requires transforming all volume elements, including empty ones, increasing computation if not parallelized [S2, unofficial].
- **Precision:** Not given in the sources.

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
        Transformed voxel grid (list of lists of lists of 4D homogeneous vectors).
    """
    Nx, Ny, Nz = voxel_grid.shape
    # Initialize output grid with homogeneous coordinates
    transformed_positions = [[[[[0.0, 0.0, 0.0, 1.0] for _ in range(Nz)] for _ in range(Ny)] for _ in range(Nx)]]
    
    # Step 1: Transform the first voxel (origin) using matrix multiplication
    p0_src = [0.0, 0.0, 0.0, 1.0]
    transformed_positions[0][0][0] = matrix_vector_multiply(transformation_matrix, p0_src)
    
    # Precompute incremental deltas (transformed basis vectors)
    dx = matrix_vector_multiply(transformation_matrix, [1.0, 0.0, 0.0, 0.0])
    dy = matrix_vector_multiply(transformation_matrix, [0.0, 1.0, 0.0, 0.0])
    dz = matrix_vector_multiply(transformation_matrix, [0.0, 0.0, 1.0, 0.0])
    
    # Step 2: Incrementally update the remaining voxels
    # Iterate in a predefined sequence to handle overlaps if necessary [S1]
    for z in range(Nz):
        for y in range(Ny):
            for x in range(Nx):
                if x == 0 and y == 0 and z == 0:
                    continue # Already computed
                elif x > 0:
                    # Update along X-axis using predecessor in X
                    transformed_positions[x][y][z] = vector_add(transformed_positions[x-1][y][z], dx)
                elif y > 0:
                    # Update along Y-axis using predecessor in Y
                    transformed_positions[x][y][z] = vector_add(transformed_positions[x][y-1][z], dy)
                elif z > 0:
                    # Update along Z-axis using predecessor in Z
                    transformed_positions[x][y][z] = vector_add(transformed_positions[x][y][z-1], dz)
                    
    return transformed_positions

def matrix_vector_multiply(matrix, vector):
    """Standard 4x4 matrix by 4-vector multiplication."""
    result = [0.0, 0.0, 0.0, 0.0]
    for i in range(4):
        s = 0.0
        for j in range(4):
            s += matrix[i][j] * vector[j]
        result[i] = s
    return result

def vector_add(v1, v2):
    """Component-wise addition of two 4D vectors."""
    return [v1[0] + v2[0], v1[1] + v2[1], v1[2] + v2[2], v1[3] + v2[3]]
```

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the total number of voxels, with a very low constant factor (3 additions per voxel) compared to $O(N)$ with a higher constant for matrix multiplication [S1][S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing the voxel grid [S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of voxels; performance is highly dependent on parallelization, as all voxels (including empty ones) must be processed [S2, unofficial].
- **Strengths:** Reduces arithmetic operations significantly; exploits spatial coherence; suitable for parallel architectures [S1][S2, unofficial].
- **Limitations:** Requires processing all voxels, including empty ones; potential for overlap issues if not handled with a specific ordering algorithm [S1][S2, unofficial].
- **When to use:** When hardware supports parallel processing and arithmetic operations are expensive; when the transformation is affine and the grid is dense or parallelization is feasible [S2, unofficial].
- **When not to use:** When the grid is sparse and most voxels are empty (wastes computation) [S2, unofficial].

## Background
The Incremental Geometric Transformation technique was developed as part of the Voxblt Engine, a voxel frame buffer processor, in the late 1980s [S1]. It was designed to perform geometric operations (translation, scaling, rotation) efficiently by exploiting inter- and intra-scan-line coherency [S1]. The method was implemented in hardware, utilizing parallel memory and barrel shifters to accelerate the incremental updates [S1]. It is also discussed in the context of volume visualization and rendering techniques, where it is noted for its suitability to parallel architectures [S2, unofficial].

## Sources
[S1] The voxblt Engine: A Voxel Frame Buffer Processor, https://doi.org/10.2312/eggh/eggh88/085-102, fetched 2026-10-26
[S2] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
