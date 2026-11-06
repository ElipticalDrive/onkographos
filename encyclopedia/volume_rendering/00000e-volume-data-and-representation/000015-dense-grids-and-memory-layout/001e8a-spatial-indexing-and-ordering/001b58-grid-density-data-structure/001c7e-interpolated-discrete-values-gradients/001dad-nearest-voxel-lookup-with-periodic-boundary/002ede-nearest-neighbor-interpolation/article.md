# Nearest Neighbor Interpolation

Also called: Nearest Voxel Lookup, Nearest-Neighbor Sampling

## Overview
Nearest Neighbor Interpolation estimates the value at a continuous sample point by retrieving the value of the single grid voxel whose center is closest to that point [S1, S2]. It serves as the simplest form of spatial sampling in volume rendering, providing a computationally efficient but low-fidelity approximation of the underlying scalar field [S1]. Unlike higher-order methods such as trilinear interpolation, it does not blend neighboring values, resulting in a piecewise-constant representation that preserves sharp discontinuities but introduces aliasing artifacts [S2].

## How it works
**Inputs:**
- A 3D grid of voxel values $V$ with dimensions $D_x \times D_y \times D_z$.
- A continuous sample position $P = (p_x, p_y, p_z)$ in normalized grid coordinates (typically $[0, D_x-1] \times [0, D_y-1] \times [0, D_z-1]$ or $[0,1]^3$ depending on normalization).

**Outputs:**
- The scalar value $V_{sample}$ associated with the nearest voxel.

**Data Structures:**
- The voxel grid $V$ is typically stored as a dense 3D array. For implementation, this is often flattened into a 1D array or accessed via stride-based indexing.
- Index coordinates $(i, j, k)$ are integers representing the voxel location.

**Steps:**
1. **Coordinate Mapping:** If $P$ is in texture space $[0,1]^3$, map it to grid coordinates by multiplying by the grid dimensions minus one (or just dimensions, depending on whether coordinates are center-based or corner-based). Let the resulting continuous coordinates be $(c_x, c_y, c_z)$.
2. **Rounding:** Round each continuous coordinate to the nearest integer to determine the voxel indices $(i, j, k)$.
   $$ i = \text{round}(c_x) $$
   $$ j = \text{round}(c_y) $$
   $$ k = \text{round}(c_z) $$
   Where $\text{round}(x)$ returns the nearest integer to $x$. In many implementations, this is achieved by adding 0.5 and truncating: $\lfloor x + 0.5 \rfloor$.
3. **Boundary Handling:** Check if $(i, j, k)$ are within the valid grid bounds $[0, D_x-1] \times [0, D_y-1] \times [0, D_z-1]$.
   - If the parent method [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) is used, apply modulo arithmetic: $i = i \pmod{D_x}$, etc.
   - If clamping is required, clamp indices to the nearest valid boundary.
4. **Lookup:** Retrieve the value $V_{sample} = V[i][j][k]$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nearest_neighbor_interpolation(grid, dims, point):
    """
    grid: 3D array of voxel values
    dims: tuple (dx, dy, dz) of grid dimensions
    point: tuple (px, py, pz) in continuous grid coordinates
    """
    # Map continuous point to integer indices
    # Assuming point is in range [0, dims-1] representing centers
    ix = int(point[0] + 0.5)
    iy = int(point[1] + 0.5)
    iz = int(point[2] + 0.5)
    
    # Boundary handling (Clamping example, replace with periodic if needed)
    ix = max(0, min(ix, dims[0] - 1))
    iy = max(0, min(iy, dims[1] - 1))
    iz = max(0, min(iz, dims[2] - 1))
    
    return grid[ix][iy][iz]
```

**Parameters:**
- Rounding mode: Standard nearest integer rounding is used [S2].
- Boundary condition: Depends on the parent context; periodic wrapping is common in texture mapping contexts [S1573], while clamping is common in scientific visualization [G].

**Edge Cases and Pitfalls:**
- **Imprecision:** The method is inherently imprecise and provides only a rough estimate of the volume's nature [S1, S2].
- **Aliasing:** Because it does not average neighboring values, it is susceptible to aliasing artifacts, particularly at low sampling rates [S2].
- **Discontinuities:** The output is discontinuous across voxel boundaries, which can cause visual "blockiness" [S1].

## Subtopics

### [Nearest Integer Rounding](4233-nearest-integer-rounding.md)
The fundamental arithmetic operation used to map continuous coordinates to discrete indices. Choose this when understanding the low-level coordinate transformation is required.

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Extends nearest neighbor lookup by handling out-of-bounds indices via modulo arithmetic. Choose this when the volume data is assumed to be periodic or when wrapping is desired.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. It involves a constant number of arithmetic operations and a single memory access.
- **Memory Complexity:** $O(1)$ auxiliary space. No additional buffers are required beyond the grid itself.
- **Performance:** It is the fastest interpolation method available [S1]. It is suitable for real-time applications where visual fidelity is secondary to speed, or for initial previews [S2].
- **Strengths:** Computational efficiency; simplicity of implementation; preservation of sharp edges without blurring.
- **Limitations:** Low visual quality; aliasing artifacts; lack of smoothness; not suitable for high-quality rendering or gradient-based lighting calculations without significant sampling [S1, S2].
- **When to use:** Real-time previews, low-resolution data, or when computational resources are severely constrained.
- **When not to use:** High-quality volume rendering, scenes requiring smooth gradients, or when aliasing artifacts are unacceptable.

## Background
Nearest neighbor interpolation is a foundational technique in computer graphics and scientific visualization. It is described in early volume rendering literature as a simplistic alternative to more complex schemes like trilinear interpolation or inverse weighting [S1]. The method is often introduced in educational contexts to contrast with higher-order interpolation methods, highlighting the trade-off between computational cost and visual accuracy [S2]. It remains relevant in texture mapping and fast approximation algorithms.

## Sources
[S1] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
[S2] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
