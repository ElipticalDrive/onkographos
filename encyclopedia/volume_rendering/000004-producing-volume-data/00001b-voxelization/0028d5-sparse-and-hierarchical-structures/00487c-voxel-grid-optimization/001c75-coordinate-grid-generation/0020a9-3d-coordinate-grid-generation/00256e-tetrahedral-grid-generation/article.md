# Tetrahedral Grid Generation

Also called: Cube to Tetrahedra Conversion, Tetrahedralization of Uniform Grid

## Overview
Tetrahedral Grid Generation converts a standard 3D voxel grid (a lattice of cubes) into a conforming mesh of tetrahedra. This method solves the problem of creating a non-orthogonal sampling domain for Signed Distance Function (SDF) evaluation or volume rendering, which can reduce aliasing artifacts compared to axis-aligned voxel grids. It sets itself apart from sibling methods like [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md) by providing a deterministic, structured connectivity map rather than stochastic point distributions, and from [Plane Grid Generation](5070-plane-grid-generation.md) by operating in full 3D volume rather than 2D slices.

## How it works
**Inputs:**
- Grid dimensions: $N_x, N_y, N_z$ (number of vertices along each axis).
- Implicit assumption: Vertices are indexed linearly in row-major order (x-fast, y-medium, z-slow).

**Outputs:**
- A tetrahedron definition matrix $TF$ of size $((N_x-1) \times (N_y-1) \times (N_z-1) \times 6, 4)$, where each row contains the four vertex indices of a tetrahedron.

**Data Structures:**
- $TF$: A 2D array or list of lists. Each element is a tuple/list of 4 integers representing vertex indices.

**Algorithm Steps:**
1. Initialize $TF$ with zeros. The total number of cubes in the grid is $N_{cubes} = (N_x-1)(N_y-1)(N_z-1)$. Since each cube is split into 6 tetrahedra, the total number of rows is $6 \times N_{cubes}$.
2. Iterate through each cube in the grid using indices $i \in [0, N_x-2]$, $j \in [0, N_y-2]$, and $k \in [0, N_z-2]$.
3. For each cube, calculate the linear indices of its 8 vertices. Let $v_1$ through $v_8$ be the vertex indices. The mapping from grid coordinates $(x,y,z)$ to linear index $idx$ is:
   $$ idx(x, y, z) = x + N_x \cdot (y + N_y \cdot z) $$
   The 8 vertices of the cube at $(i,j,k)$ are:
   - $v_1 = idx(i, j, k)$
   - $v_2 = idx(i, j+1, k)$
   - $v_3 = idx(i+1, j, k)$
   - $v_4 = idx(i+1, j+1, k)$
   - $v_5 = idx(i, j, k+1)$
   - $v_6 = idx(i, j+1, k+1)$
   - $v_7 = idx(i+1, j, k+1)$
   - $v_8 = idx(i+1, j+1, k+1)$
4. Define the 6 tetrahedra for this cube by assigning the vertex indices to the next 6 rows of $TF$ (indices $t_i$ to $t_i+5$):
   - Tetrahedron 1: $(v_1, v_3, v_8, v_7)$
   - Tetrahedron 2: $(v_1, v_8, v_5, v_7)$
   - Tetrahedron 3: $(v_1, v_3, v_4, v_8)$
   - Tetrahedron 4: $(v_1, v_4, v_2, v_8)$
   - Tetrahedron 5: $(v_1, v_6, v_5, v_8)$
   - Tetrahedron 6: $(v_1, v_2, v_6, v_8)$
5. Increment the row counter in $TF$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_tetrahedral_grid(nx, ny, nz):
    """
    Generates a tetrahedral grid from a voxel grid of size nx x ny x nz.
    
    Args:
        nx, ny, nz: Integers, number of vertices along each axis.
        
    Returns:
        tf: List of lists, each inner list contains 4 vertex indices.
    """
    # Calculate total number of cubes and tetrahedra
    num_cubes = (nx - 1) * (ny - 1) * (nz - 1)
    num_tets = num_cubes * 6
    
    # Initialize output matrix
    tf = [[0, 0, 0, 0] for _ in range(num_tets)]
    
    # Helper for linear indexing
    def linear_index(x, y, z):
        return x + nx * (y + ny * z)
    
    row_idx = 0
    
    # Iterate over each cube in the grid
    for i in range(nx - 1):
        for j in range(ny - 1):
            for k in range(nz - 1):
                # Calculate vertex indices for the current cube
                v1 = linear_index(i, j, k)
                v2 = linear_index(i, j + 1, k)
                v3 = linear_index(i + 1, j, k)
                v4 = linear_index(i + 1, j + 1, k)
                v5 = linear_index(i, j, k + 1)
                v6 = linear_index(i, j + 1, k + 1)
                v7 = linear_index(i + 1, j, k + 1)
                v8 = linear_index(i + 1, j + 1, k + 1)
                
                # Assign 6 tetrahedra to the output matrix
                # Tet 1: (v1, v3, v8, v7)
                tf[row_idx] = [v1, v3, v8, v7]
                row_idx += 1
                
                # Tet 2: (v1, v8, v5, v7)
                tf[row_idx] = [v1, v8, v5, v7]
                row_idx += 1
                
                # Tet 3: (v1, v3, v4, v8)
                tf[row_idx] = [v1, v3, v4, v8]
                row_idx += 1
                
                # Tet 4: (v1, v4, v2, v8)
                tf[row_idx] = [v1, v4, v2, v8]
                row_idx += 1
                
                # Tet 5: (v1, v6, v5, v8)
                tf[row_idx] = [v1, v6, v5, v8]
                row_idx += 1
                
                # Tet 6: (v1, v2, v6, v8)
                tf[row_idx] = [v1, v2, v6, v8]
                row_idx += 1
                
    return tf
```

**Edge Cases and Pitfalls:**
- **Parallelization:** Parallelizing the outer loops requires careful handling of the output array index to avoid race conditions, as multiple threads may attempt to write to $TF$ simultaneously if not synchronized or if using atomic increments [S1, unofficial].
- **Boundary Conditions:** The loop ranges $0$ to $N-2$ ensure that vertex indices $i+1, j+1, k+1$ do not exceed $N-1$, which is the maximum valid index for a grid of size $N$.

## Subtopics

### Polyhedral Cell Definition
In multi-resolution tetrahedral encoders (e.g., TetraSDF), the grid is not uniform but hierarchical. The encoder-induced polyhedral cell $\mathcal{C}_{\mathbf{x}}$ for a point $\mathbf{x}$ is defined as the intersection of the tetrahedra $\mathcal{T}^{(\ell)}(\mathbf{x})$ containing $\mathbf{x}$ across all resolution levels $\ell \in \{0, \dots, L-1\}$ [S2].
$$\mathcal{C}_{\mathbf{x}} := \bigcap_{\ell=0}^{L-1} \mathcal{T}^{(\ell)}(\mathbf{x}) \quad (45)$$
The geometric progression ratio $\gamma$ determines the resolution scaling:
$$\gamma := \exp\left(\frac{\ln N_{\max} - \ln N_{\min}}{L - 1}\right) \quad (47)$$
If $\gamma \in \mathbb{N}$ and $\gamma \geq 2$, every cell $\mathcal{C}_{\mathbf{x}}$ is a tetrahedron at the finest level [S2].
**Pitfall:** For $L > 1$, the encoder $\tau$ is piecewise affine and continuous. It is affine within each cell $\mathcal{C}_{\mathbf{x}}$, not just within individual tetrahedra of the finest level [S2].

## Operating characteristics
**Time Complexity:** $O(N_x N_y N_z)$, as each cube is processed exactly once.
**Memory Complexity:** $O(N_x N_y N_z)$ for storing the tetrahedron indices. The storage is $6 \times (N_x-1)(N_y-1)(N_z-1)$ tuples of 4 integers.
**Strengths:**
- Provides a conforming mesh suitable for finite element methods or marching tetrahedra isosurface extraction.
- Deterministic and easy to implement.
- Reduces anisotropy compared to axis-aligned voxel grids.
**Limitations:**
- Connectivity is fixed; cannot adapt to local feature density without multi-resolution variants.
- Parallelization requires synchronization for the output buffer.
**When to use:** When a structured, non-orthogonal sampling grid is required for SDF evaluation or volume rendering, and the geometry is relatively uniform.
**When not to use:** When adaptive resolution is needed (use [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md)) or when stochastic sampling is preferred for noise reduction.

## Background
The method of decomposing a cube into 6 tetrahedra is a standard technique in computational geometry and finite element analysis. In the context of volume rendering and SDFs, it is used to facilitate isosurface extraction algorithms like Marching Tetrahedra, which avoid the ambiguity issues of Marching Cubes. The specific vertex ordering $(v_1, v_3, v_8, v_7)$, etc., is one of several valid decompositions that ensure the tetrahedra fill the cube without overlap. The implementation details provided here are derived from practical repositories for marching tetrahedra reconstruction [S1, unofficial] and theoretical frameworks for multi-resolution tetrahedral encoders [S2].

## Sources
[S1] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
[S2] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
