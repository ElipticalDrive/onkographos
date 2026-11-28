# Tetrahedral Grid Generation

Also called: Cube to Tetrahedra Conversion, Tetrahedralization of Uniform Grid

## Overview
Tetrahedral Grid Generation converts a standard 3D voxel grid into a conforming mesh of tetrahedra [G]. This method provides a deterministic, structured connectivity map for sampling domains [G]. It differs from [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md) by avoiding stochastic distributions, and from [Plane Grid Generation](5070-plane-grid-generation.md) by operating in full 3D volume [G].

## How it works
**Inputs:**
- Grid dimensions $N_x, N_y, N_z$ representing the number of vertices along each axis [S1, unofficial].
- Implicit assumption: Vertices are indexed linearly in row-major order (x-fast, y-medium, z-slow) [S1, unofficial].

**Outputs:**
- A tetrahedron definition matrix $TF$ of size $((N_x-1) \times (N_y-1) \times (N_z-1) \times 6, 4)$, where each row contains four vertex indices [S1, unofficial].

**Data Structures:**
- $TF$: A 2D array where each element is a tuple of four integers [G].

**Algorithm Steps:**
1. Initialize $TF$ with zeros. The total number of cubes is $N_{cubes} = (N_x-1)(N_y-1)(N_z-1)$ [G]. Since each cube splits into 6 tetrahedra, $TF$ has $6 \times N_{cubes}$ rows [S1, unofficial].
2. Iterate through each cube using indices $i \in [0, N_x-2]$, $j \in [0, N_y-2]$, and $k \in [0, N_z-2]$ [S1, unofficial].
3. Calculate the linear indices of the 8 vertices for the current cube using $idx(x, y, z) = x + N_x \cdot (y + N_y \cdot z)$ [S1, unofficial]. The vertices are $v_1$ through $v_8$ [S1, unofficial].
4. Define the 6 tetrahedra by assigning vertex indices to rows $t_i$ through $t_i+5$ of $TF$: $(v_1,v_3,v_8,v_7)$, $(v_1,v_8,v_5,v_7)$, $(v_1,v_3,v_4,v_8)$, $(v_1,v_4,v_2,v_8)$, $(v_1,v_6,v_5,v_8)$, $(v_1,v_2,v_6,v_8)$ [S1, unofficial].
5. Increment the row counter in $TF$ [G].

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
- **Boundary Conditions:** The loop ranges $0$ to $N-2$ ensure that vertex indices $i+1, j+1, k+1$ do not exceed $N-1$, which is the maximum valid index for a grid of size $N$ [G].

## Subtopics

### Polyhedral Cell Definition
In multi-resolution tetrahedral encoders, the grid is hierarchical rather than uniform [S2]. The encoder-induced polyhedral cell $\mathcal{C}_{\mathbf{x}}$ for a point $\mathbf{x}$ is defined as the intersection of the tetrahedra $\mathcal{T}^{(\ell)}(\mathbf{x})$ containing $\mathbf{x}$ across all resolution levels $\ell \in \{0, \dots, L-1\}$ [S2].
$$\mathcal{C}_{\mathbf{x}} := \bigcap_{\ell=0}^{L-1} \mathcal{T}^{(\ell)}(\mathbf{x}) \quad (45)$$
The geometric progression ratio $\gamma$ determines the resolution scaling:
$$\gamma := \exp\left(\frac{\ln N_{\max} - \ln N_{\min}}{L - 1}\right) \quad (47)$$
If $\gamma \in \mathbb{N}$ and $\gamma \geq 2$, every cell $\mathcal{C}_{\mathbf{x}}$ is a tetrahedron at the finest level [S2].
**Pitfall:** For $L > 1$, the encoder $\tau$ is piecewise affine and continuous. It is affine within each cell $\mathcal{C}_{\mathbf{x}}$, not just within individual tetrahedra of the finest level [S2].

## Operating characteristics
**Time Complexity:** $O(N_x N_y N_z)$, as each cube is processed exactly once [G].
**Memory Complexity:** $O(N_x N_y N_z)$ for storing the tetrahedron indices. The storage is $6 \times (N_x-1)(N_y-1)(N_z-1)$ tuples of 4 integers [S1, unofficial].
**Strengths:**
- Provides a conforming mesh suitable for finite element methods or marching tetrahedra isosurface extraction [G].
- Deterministic and easy to implement [G].
- Reduces anisotropy compared to axis-aligned voxel grids [G].
**Limitations:**
- Connectivity is fixed; cannot adapt to local feature density without multi-resolution variants [G].
- Parallelization requires synchronization for the output buffer [S1, unofficial].
**When to use:** When a structured, non-orthogonal sampling grid is required for SDF evaluation or volume rendering, and the geometry is relatively uniform [G].
**When not to use:** When adaptive resolution is needed (use [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md)) or when stochastic sampling is preferred for noise reduction [G].

## Background
The method of decomposing a cube into 6 tetrahedra is a standard technique in computational geometry and finite element analysis [G]. In the context of volume rendering and SDFs, it is used to facilitate isosurface extraction algorithms like Marching Tetrahedra, which avoid the ambiguity issues of Marching Cubes [G]. The specific vertex ordering $(v_1, v_3, v_8, v_7)$, etc., is one of several valid decompositions that ensure the tetrahedra fill the cube without overlap [G]. The implementation details provided here are derived from practical repositories for marching tetrahedra reconstruction [S1, unofficial] and theoretical frameworks for multi-resolution tetrahedral encoders [S2].

## Sources
[S1] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
[S2] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
