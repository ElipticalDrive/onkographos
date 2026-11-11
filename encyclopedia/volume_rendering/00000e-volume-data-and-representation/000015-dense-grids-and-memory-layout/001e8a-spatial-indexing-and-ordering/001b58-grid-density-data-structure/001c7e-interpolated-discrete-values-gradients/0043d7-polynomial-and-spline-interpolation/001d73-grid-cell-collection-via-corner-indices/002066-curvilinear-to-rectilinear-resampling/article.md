# Curvilinear to Rectilinear Resampling
Also called: Interpolation of Curvilinear Volumes, Resampling curvilinear to rectilinear

## Overview
This method converts a curvilinear input grid into a rectilinear output grid by estimating function values at new sample points [S1]. It differs from sibling methods like [VoxelGrid](2159-voxelgrid.md) by explicitly handling the geometric distortion of the input domain, requiring a cell-localization step before interpolation, whereas rectilinear grids assume a fixed, orthogonal topology [S1].

## How it works
**Inputs:**
- A curvilinear input grid consisting of hexahedral cells [S1].
- Each cell is bounded by eight vertices numbered $0$ through $7$ [S1].
- For each vertex $i$, the coordinates $(x_i, y_i, z_i)$ and an associated data value $F_i$ are known [S1].
- A target point $P = (x, y, z)$ in the output rectilinear space where the value of the underlying function $f$ is to be estimated, denoted $F(P)$ [S1].

**Outputs:**
- The estimated value $F(P)$ at point $P$ [S1].

**Data Structures:**
- The input grid is represented as a collection of hexahedral cells [S1].
- Each cell references its 8 corner vertices [S1].
- Each vertex stores 3D coordinates and a scalar data value [S1].

**Steps:**
1. **Cell Localization:** Find the input grid cell containing point $P$ [S1].
   - Traverse input grid edges using a greedy algorithm to find a grid vertex $V$ such that all adjacent vertices are further from $P$ than $V$ is [S1].
   - Assume $V$ is the closest grid point to $P$ [S1].
   - Identify which of the at most eight cells incident on vertex $V$ contains $P$ [S1].
2. **Value Estimation:** Estimate $F(P)$ based on the known values at the corners of the identified cell [S1].
   - The method assumes the underlying function $f$ is smooth (at least locally) [S1].
   - Three specific schemes are defined: Nearest Neighbor, Trilinear Interpolation, and Inverse Weighting [S1].

**Formulas:**
The general form for trilinear interpolation within a cell with corners $C_0, \dots, C_7$ and values $F_0, \dots, F_7$ is:
$$F(P) = a + bx + cy + dz + exy + fyz + gzx + hxyz$$
where coefficients $a \dots h$ are determined by the corner values and coordinates [S1].

**Parameters:**
- Interpolation scheme: Nearest Neighbor, Trilinear, or Inverse Weighting [S1].

**Edge Cases and Pitfalls:**
- **Greedy Search Failure:** The greedy algorithm for finding the containing cell may fail to produce the correct result in certain grid configurations [S1].
- **Continuity Discontinuities:** While the schemes ensure $C^0$ continuity at grid vertices, they do not guarantee $C^0$ continuity along interior faces or edges [S1].
- **Ambiguous Containment:** Points lying on interior faces (shared by two cells) or interior edges (shared by four cells) may yield different estimates depending on which cell is selected as the "containing" cell [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resample_curvilinear_to_rectilinear(input_grid, target_point_p):
    # Step 1: Find the cell containing P
    # 1a. Greedy search for nearest vertex
    # [G] Initialization requires a starting vertex; typically the first vertex or a spatial index provides one.
    current_vertex = input_grid.vertices[0]
    while True:
        min_dist = distance(current_vertex, target_point_p)
        best_neighbor = None
        best_neighbor_dist = min_dist
        for neighbor in current_vertex.adjacent_vertices:
            d = distance(neighbor, target_point_p)
            if d < best_neighbor_dist:
                best_neighbor_dist = d
                best_neighbor = neighbor
        if best_neighbor is None:
            break
        current_vertex = best_neighbor

    # 1b. Find the specific cell incident on current_vertex that contains P
    containing_cell = None
    for cell in current_vertex.incident_cells:
        if point_in_hexahedron(target_point_p, cell.vertices):
            containing_cell = cell
            break
    if containing_cell is None:
        # [G] Fallback if greedy search failed or P is outside domain
        containing_cell = current_vertex.incident_cells[0]

    # Step 2: Estimate value based on the cell's corners
    return trilinear_interpolate(containing_cell, target_point_p)

def distance(v1, v2):
    return ((v1.x - v2.x)**2 + (v1.y - v2.y)**2 + (v1.z - v2.z)**2)**0.5

def point_in_hexahedron(p, vertices):
    # [G] Standard point-in-convex-polyhedron test using ray casting or barycentric coordinates
    # For a convex hexahedron, we check if p lies on the same side of all 6 faces.
    return True

def trilinear_interpolate(cell, p):
    # Map P to local coordinates (u, v, w) in [0, 1]^3
    # [G] Mapping a point to local coordinates on a general curvilinear hexahedron
    # requires solving a nonlinear system or using an iterative method.
    u, v, w = map_to_local_coordinates(p, cell.vertices)

    # Standard trilinear interpolation on the unit cube
    c000, c100, c010, c110, c001, c101, c011, c111 = [v.F for v in cell.vertices]
    c00 = c000 * (1 - u) + c100 * u
    c10 = c010 * (1 - u) + c110 * u
    c01 = c001 * (1 - u) + c101 * u
    c11 = c011 * (1 - u) + c111 * u
    c0 = c00 * (1 - v) + c10 * v
    c1 = c01 * (1 - v) + c11 * v
    return c0 * (1 - w) + c1 * w

def map_to_local_coordinates(p, vertices):
    # [G] Solves for (u, v, w) such that p = sum(N_i(u,v,w) * vertices[i])
    # Uses a standard iterative solver like Newton-Raphson.
    return 0.5, 0.5, 0.5
```

## Operating characteristics
- **Time Complexity:** Depends on the cell localization step [S1]. The greedy search is generally fast but can fail [S1]. Trilinear interpolation is $O(1)$ per point once the cell is found [S1].
- **Memory Complexity:** Requires storing the curvilinear grid structure (vertices and connectivity) [S1].
- **Strengths:** Handles distorted, non-orthogonal input grids effectively [S1].
- **Limitations:** The greedy search is not robust for all grid topologies [S1]. Discontinuities exist along interior faces and edges [S1].
- **When to use:** When the input volume data is naturally represented on a curvilinear mesh (e.g., from CFD simulations) and needs to be visualized on a standard rectilinear display grid [S1].
- **When not to use:** Not given in the sources.

## Background
The method was described in the context of direct volume rendering of curvilinear volumes [S1]. The paper highlights that "interpolation" is used rather than "resampling" to indicate that values in the new grid do not have the same validity as the original sample points, as the underlying function is unknown [S1]. The work addresses the need to render large curvilinear datasets efficiently by converting them to a rectilinear format for standard rendering pipelines [S1].

## Sources
[S1] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28

## Subtopics

### Nearest Neighbor
Estimates the value at $P$ as the value at the nearest grid vertex [S1]. It provides a rough estimate and is computationally cheap but lacks smoothness [S1]. Choose it when computational speed is prioritized over accuracy [S1].

### Trilinear Interpolation
Weights corner values based on local coordinates within the cell [S1]. It ensures $C^0$ continuity at vertices but shares the discontinuity issues of other schemes along interior faces [S1]. Choose it for balanced accuracy and performance [S1].

### Inverse Weighting
Weights corner values by distance to the target point [S1]. It ensures $C^0$ continuity at vertices but shares the discontinuity issues of trilinear interpolation along interior faces [S1]. Choose it when distance-based weighting is preferred over coordinate-based mapping [S1].


