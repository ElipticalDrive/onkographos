# Marching Lines Algorithm
## Overview
The Marching Lines Algorithm (MLA) is a surface extraction method that generates isosurfaces by processing 3D grid cells as a series of 2D slices, effectively extending the logic of [Marching Squares](918-marching-squares.md) into three dimensions. It differs from template-based methods like [Marching Cubes](918-marching-cubes.md) by not relying on a fixed lookup table for cell configurations, instead requiring post-processing to resolve topological ambiguities that arise from its directional nature.
## How it works
The MLA operates by slicing a 3D scalar field into 2D planes and applying 2D contouring logic along specific axes.

**Inputs and Outputs:**
- Input: A 3D scalar field $V(x,y,z)$ defined on a regular grid.
- Output: A set of line segments (edges) representing the isosurface intersection with the grid cells.

**Data Structures:**
- Grid: A 3D array of scalar values.
- Edge List: A list of 3D line segments $(P_1, P_2)$ where $P_1, P_2$ are vertices on the grid edges.

**Steps:**
1. **Slicing:** Iterate through the 3D volume along one primary axis (e.g., $z$). For each $z$-slice, treat the data as a 2D grid.
2. **2D Marching:** Apply [Marching Squares](918-marching-squares.md) logic to each 2D cell in the slice to identify crossing edges. This generates 2D line segments within the slice plane.
3. **Inter-slice Connection:** Connect line segments from adjacent slices ($z$ and $z+1$) that share grid vertices or edges. This step is critical for forming continuous surfaces.
4. **Ambiguity Resolution:** Resolve "apparent cell ambiguities" where the 2D projection suggests multiple valid topologies. This requires additional logic beyond the basic 2D case, as the 3D context may not be fully captured by the 2D slice alone [S1, unofficial].

**Formulas:**
- Linear interpolation for vertex position on an edge between grid points $V_1$ and $V_2$ with values $f_1$ and $f_2$:
  $$ P = V_1 + \frac{I - f_1}{f_2 - f_1} (V_2 - V_1) $$
  where $I$ is the isovalue.

**Parameters:**
- Isovalue $I$: The scalar threshold for surface extraction.
- Axis of slicing: Typically $z$, but can be any axis.

**Edge Cases and Pitfalls:**
- **Directional Ambiguities:** The MLA suffers from directional ambiguities because it processes data in a specific order (slices), which can lead to inconsistent topologies if not handled [S1, unofficial].
- **Apparent Cell Ambiguities:** Unlike template-based methods, MLA requires further processing to properly resolve ambiguities that appear in the 2D slices but are resolved in 3D [S1, unofficial].
- **Discontinuities:** Failure to correctly connect segments between slices can result in holes or non-manifold geometry.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_lines_algorithm(grid, isovalue):
    """
    Extracts isosurface line segments from a 3D grid using the Marching Lines approach.
    """
    segments = []
    height, width, depth = grid.shape
    
    # Step 1 & 2: Slice and March
    for z in range(depth - 1):
        # Extract 2D slice
        slice_2d = grid[:, :, z]
        next_slice_2d = grid[:, :, z + 1]
        
        # Apply Marching Squares to current slice
        current_segments = marching_squares_2d(slice_2d, isovalue)
        
        # Apply Marching Squares to next slice
        next_segments = marching_squares_2d(next_slice_2d, isovalue)
        
        # Step 3: Connect segments between slices
        # This requires matching vertices from current_segments to next_segments
        # based on shared grid coordinates.
        connected_segments = connect_segments_3d(current_segments, next_segments, z, z + 1)
        segments.extend(connected_segments)
        
    # Step 4: Resolve ambiguities
    # This step is non-trivial and depends on the specific implementation of ambiguity resolution.
    # It involves checking 2x2x2 neighborhoods or similar structures to ensure topological consistency.
    resolved_segments = resolve_ambiguities(segments, grid, isovalue)
    
    return resolved_segments

def marching_squares_2d(slice_2d, isovalue):
    """
    Standard Marching Squares implementation for a 2D grid.
    Returns a list of 2D line segments.
    """
    # Implementation details of Marching Squares are assumed to be available from [Marching Squares](918-marching-squares.md).
    # This function iterates over 2D cells, determines the case index, and returns edge segments.
    pass

def connect_segments_3d(segments_z, segments_z1, z, z1):
    """
    Connects 2D segments from slice z to slice z1.
    """
    # Logic to match endpoints of segments in segments_z with those in segments_z1
    # if they share grid vertices.
    pass

def resolve_ambiguities(segments, grid, isovalue):
    """
    Resolves apparent cell ambiguities.
    """
    # This function implements the specific logic to handle the directional ambiguities
    # mentioned in [S1, unofficial].
    pass
```
## Subtopics
## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of grid cells, similar to Marching Cubes, but with potential overhead for ambiguity resolution.
- **Memory Complexity:** $O(N)$ for storing the grid and output segments.
- **Strengths:** Simpler conceptual basis by reducing 3D to 2D problems.
- **Limitations:** Requires complex post-processing for ambiguity resolution; not template-based, making it less straightforward to implement than Marching Cubes.
- **When to use:** When 2D processing pipelines are already in place or when adaptive slicing is beneficial.
- **When not to use:** When topological robustness is critical and template-based methods are preferred.
## Background
The Marching Lines Algorithm is part of the family of surface extraction methods that extend 2D contouring techniques to 3D. It is discussed in the context of Volume-Enclosing Surface Extraction (VESTA) and related methods [S1, unofficial]. The method is noted for its non-template-based nature and the need for additional processing to handle ambiguities [S1, unofficial].
## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
