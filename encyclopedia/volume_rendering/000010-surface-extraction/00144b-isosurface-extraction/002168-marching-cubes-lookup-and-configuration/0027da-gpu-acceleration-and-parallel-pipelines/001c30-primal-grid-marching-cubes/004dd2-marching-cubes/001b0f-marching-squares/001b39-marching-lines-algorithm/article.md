# Marching Lines Algorithm
## Overview
The Marching Lines Algorithm (MLA) extracts isosurfaces from 3D scalar fields by processing the volume as a sequence of 2D slices, extending 2D contouring to three dimensions [G]. It differs from template-based methods like Marching Cubes by not relying on a fixed lookup table, instead requiring post-processing to resolve apparent cell ambiguities [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A 3D scalar field $V$ defined on a regular grid, and an isovalue $I$.
- Output: A list of 3D line segments representing the isosurface intersection with the grid cells [G].

**Data Structures:**
- Grid: A 3D array of scalar values [G].
- Edge List: A list of 3D line segments $(P_1, P_2)$ where $P_1, P_2$ are interpolated vertices on the grid edges [G].

**Steps:**
1. **Slicing:** Iterate through the 3D volume along a chosen axis (e.g., $z$) [G].
2. **2D Marching:** Apply 2D marching squares logic to each slice to identify crossing edges [G].
3. **Inter-slice Connection:** Connect line segments from adjacent slices that share grid vertices [G].
4. **Ambiguity Resolution:** Resolve apparent cell ambiguities where the 2D projection suggests multiple valid topologies [S1, unofficial].

**Formulas:**
- Linear interpolation for vertex position on an edge between grid points $V_1$ and $V_2$ with values $f_1$ and $f_2$:
  $$ P = V_1 + \frac{I - f_1}{f_2 - f_1} (V_2 - V_1) $$
  where $I$ is the isovalue [G].

**Parameters:**
- Isovalue $I$: The scalar threshold for surface extraction [G].
- Axis of slicing: Typically $z$, but can be any axis [G].

**Edge Cases and Pitfalls:**
- **Apparent Cell Ambiguities:** Unlike template-based methods, MLA requires further processing to properly resolve ambiguities that appear in the 2D slices but are resolved in 3D [S1, unofficial].
- **Discontinuities:** Failure to correctly connect segments between slices can result in holes or non-manifold geometry [G].
- **Precision:** Floating-point comparisons for vertex matching between slices require tolerance checks to avoid missed connections [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_lines_algorithm(grid, isovalue):
    segments = []
    H, W, D = grid.shape
    for z in range(D - 1):
        segs_curr = marching_squares_2d(grid[:, :, z], isovalue)
        segs_next = marching_squares_2d(grid[:, :, z + 1], isovalue)
        # Connect segments between adjacent slices sharing grid vertices
        for s1 in segs_curr:
            for s2 in segs_next:
                if s1.end == s2.start:
                    segments.append((s1.start, s2.end))
    # Sources do not specify the exact ambiguity resolution algorithm; standard approaches check 2x2x2 neighborhoods or use topological consistency checks [G].
    return segments

def marching_squares_2d(grid_2d, isovalue):
    segs = []
    H, W = grid_2d.shape
    for y in range(H - 1):
        for x in range(W - 1):
            v0, v1, v2, v3 = grid_2d[y, x], grid_2d[y, x+1], grid_2d[y+1, x+1], grid_2d[y+1, x]
            case = (v0 > isovalue) + 2*(v1 > isovalue) + 4*(v2 > isovalue) + 8*(v3 > isovalue)
            # Linear interpolation for vertex position on an edge
            def interp(p1, p2, f1, f2):
                return (p1 + (isovalue - f1) / (f2 - f1) * (p2 - p1))
            p_top = interp((x, y), (x+1, y), v0, v1)
            p_bot = interp((x, y+1), (x+1, y+1), v2, v3)
            p_left = interp((x, y), (x, y+1), v0, v2)
            p_right = interp((x+1, y), (x+1, y+1), v1, v3)
            if case in [1, 2, 4, 7, 8, 11, 13, 14]:
                segs.append((p_top, p_bot))
            elif case in [3, 5, 6, 9, 10, 12]:
                segs.append((p_left, p_right))
    return segs
```
## Subtopics
Cubical Marching Squares (CMS) is an adaptive surface extraction method that converts 3D Hermite data into a polygonal mesh by unfolding each cube into six 2D faces, applying Marching Squares to generate isocurves, and reconnecting these segments in 3D [S1, unofficial]. It eliminates inter-cell dependency and crack patching by sampling sharp features on faces, allowing for consistent adaptive subdivision without the edge-flipping artifacts found in methods like Edge Matching Cubes (EMC) [S1, unofficial]. See [Cubical Marching Squares](2780-cubical-marching-squares.md).
## Operating characteristics
Not given in the sources.
## Background
The MLA is discussed in the context of Volume-Enclosing Surface Extraction (VESTA) and related surface extraction techniques [S1, unofficial]. It is noted for its non-template-based nature and the necessity of post-processing to handle ambiguities [S1, unofficial].
## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
