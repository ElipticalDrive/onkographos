# Voxel Grid Coordinate Mapping

Also called: Grid Index to World Coordinate Mapping, Linear Buffer Index to World Coordinate Mapping

## Overview
Voxel Grid Coordinate Mapping defines the bijective transformations between discrete voxel indices, linear buffer positions, and continuous world-space coordinates. It solves the problem of locating specific volumetric data elements in physical space for ray casting or shading. It is distinguished from [Voxel Center Computation from Indices](11239-voxel-center-computation-from-indices.md) by explicitly handling the linear buffer indexing scheme (row-major flattening) and the specific vertex coordinate generation required for geometry-based volume rendering, rather than just center point calculation.

## How it works
**Inputs:**
- `SamplesPerSide`: Integer, the number of voxels along one axis of the cubic grid.
- `CellSize`: Float, the physical width/height/depth of a single voxel.
- `index`: Integer, the linear buffer index.
- `(x, y, z)`: Integers, the 3D grid coordinates.

**Outputs:**
- `(x, y, z)`: 3D grid coordinates.
- `index`: Linear buffer index.
- `(xPos, yPos, zPos)`: World-space vertex coordinates.

**Data Structures:**
The linear buffer is a 1D array of size `SamplesPerSide^3`. The mapping assumes a row-major layout where the Z-axis is the slowest varying dimension, Y is intermediate, and X is the fastest.

**Steps:**

1. **Linear Index to 3D Grid Coordinates:**
   Given a linear `index`, compute the 3D grid coordinates `(x, y, z)` using integer division and modulo operations [S1, unofficial]:
   $$
   \begin{aligned}
   z &= \text{index} / (\text{SamplesPerSide} \times \text{SamplesPerSide}) \\
   y &= (\text{index} \pmod{\text{SamplesPerSide} \times \text{SamplesPerSide}}) / \text{SamplesPerSide} \\
   x &= \text{index} \pmod{\text{SamplesPerSide}}
   \end{aligned}
   $$

2. **3D Grid Coordinates to Linear Index:**
   Given `(x, y, z)`, compute the linear `index` [S1, unofficial]:
   $$
   \text{index} = x + y \times \text{SamplesPerSide} + z \times \text{SamplesPerSide} \times \text{SamplesPerSide}
   $$

3. **3D Grid Coordinates to World Space Vertex Coordinates:**
   Map the discrete grid indices to continuous world coordinates. First, calculate the bounding half-extent `bound` based on the number of intervals (`gridSize = SamplesPerSide - 1`) [S1, unofficial]:
   $$
   \text{gridSize} = \text{SamplesPerSide} - 1
   $$
   $$
   \text{bound} = (\text{gridSize} / 2) \times \text{CellSize}
   $$
   Then, compute the world coordinates using linear interpolation (Lerp) from `-bound` to `bound` [S1, unofficial]:
   $$
   \begin{aligned}
   x_{\text{Pos}} &= \text{Lerp}(-\text{bound}, \text{bound}, x / \text{gridSize}) \\
   y_{\text{Pos}} &= \text{Lerp}(-\text{bound}, \text{bound}, y / \text{gridSize}) \\
   z_{\text{Pos}} &= \text{Lerp}(-\text{bound}, \text{bound}, z / \text{gridSize})
   \end{aligned}
   $$
   Where $\text{Lerp}(a, b, t) = a + t(b - a)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_to_grid(index: int, samples_per_side: int) -> tuple[int, int, int]:
    """Converts a linear buffer index to 3D grid coordinates (x, y, z)."""
    z = index // (samples_per_side * samples_per_side)
    y = (index % (samples_per_side * samples_per_side)) // samples_per_side
    x = index % samples_per_side
    return (x, y, z)

def grid_to_linear(x: int, y: int, z: int, samples_per_side: int) -> int:
    """Converts 3D grid coordinates to a linear buffer index."""
    return x + y * samples_per_side + z * samples_per_side * samples_per_side

def grid_to_world(x: int, y: int, z: int, cell_size: float, samples_per_side: int) -> tuple[float, float, float]:
    """Maps 3D grid indices to world-space vertex coordinates."""
    grid_size = samples_per_side - 1
    bound = (grid_size / 2.0) * cell_size
    
    # Lerp function: a + t * (b - a)
    # t is normalized coordinate in [0, 1]
    t_x = x / grid_size
    t_y = y / grid_size
    t_z = z / grid_size
    
    x_pos = -bound + t_x * (2.0 * bound)
    y_pos = -bound + t_y * (2.0 * bound)
    z_pos = -bound + t_z * (2.0 * bound)
    
    return (x_pos, y_pos, z_pos)
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `SamplesPerSide` is 1, `gridSize` becomes 0, causing division by zero in the world coordinate mapping. This case is degenerate and typically invalid for volume rendering grids [S1, unofficial].
- **Identity Transformation:** If the mapping between vertex coordinates and voxel coordinates is not an identity transformation (i.e., if the geometry vertices do not align with the computed world coordinates), the rendered volume will appear warped [S2].

## Subtopics

### Per-Vertex Coordinate Mapping
This variant explicitly specifies voxel coordinates on a per-vertex basis, replacing standard geometry initialization functions (e.g., `my_InitGeometry`) to define the spatial relationship between the rendering primitives and the volume data [S2]. It is chosen when the volume needs to be rendered as explicit geometry (e.g., marching cubes or ray-marched quads) rather than via texture-based ray casting. A critical pitfall is that if the mapping between the vertex coordinates and the underlying voxel grid coordinates is not an identity transformation, the rendered volume will appear geometrically warped [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ for all mappings. The operations consist of basic arithmetic (addition, multiplication, division, modulo).
- **Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are required beyond the input parameters.
- **Strengths:** Extremely fast, deterministic, and suitable for GPU shader execution where branchless arithmetic is preferred.
- **Limitations:** Assumes a regular, axis-aligned, cubic grid. Does not handle non-uniform voxel sizes or irregular grid layouts.

## Background
The linear indexing scheme (row-major flattening) is a standard convention in computer graphics for storing 3D textures and voxel grids in 1D memory buffers. The specific world coordinate mapping formula, which centers the grid around the origin using `SamplesPerSide - 1` as the divisor, is documented in the IsoMesh project [S1, unofficial]. The concept of per-vertex coordinate mapping and the warning regarding warping due to non-identity transformations originates from the SGI Volumizer documentation [S2].

## Sources
[S1] EmmetOT/IsoMesh, https://github.com/EmmetOT/IsoMesh, fetched 2026-10-26 (unofficial)
[S2] Appendix A. Volume Rendering Examples, https://techpubs.jurassic.nl/library/manuals/3000/007-3720-002/sgi_html/apa.html, fetched 2026-10-30
