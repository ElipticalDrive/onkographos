# Discrete Polygonal Occlusion via 2D Boundary Coverage

Also called: Conditional InnerShell Emission, Couvertures discrétisées, Emission conditionnelle des innerShell selon les six voisins, VXL-044, VXL-044 Discrete Boundary Coverage, VXL-045, VXL-046, VXL-046 Conditional InnerShell Emission

## Overview
Discrete Polygonal Occlusion via 2D Boundary Coverage (VXL-044) is a synchronous mesh baking optimization that reduces polygon count by determining which inner shell polygons of a voxel cell are occluded by adjacent cells using discrete 2D coverage masks on cell boundaries [S1]. It solves the problem of replacing exact rational-coordinate geometric clipping with a deterministic, integer-based rasterization approach to verify if a boundary is fully enclosed by opaque neighbors [S1]. Unlike its siblings, which may rely on continuous geometry or per-frame GPU culling, this method operates on a fixed 1000x1000 integer grid per face during the model bake phase, ensuring deterministic results without floating-point precision issues [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- A `Volume` containing `Cell` objects, each referencing a `Shape` definition [S1].
- A `VoxelMesher` instance capable of querying neighbor states via `_outsideCell` [S1].
- For each cell, the `outerShell` polygons of the current cell and its six immediate neighbors [S1].

**Outputs:**
- A `Model` containing the final mesh [S1].
- For each cell: a boolean flag `emitInnerShell` determining whether to emit all inner shell polygons or none [S1].

### Data Structures
**OutsideCell**
Represents the state of a neighbor relative to the current cell [S1].
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct OutsideCell {
    enum class State { Empty, Occupied, Unknown };
    State state;
    // Optional reference to the neighbor Cell
};
```
- `State`:
  - `Empty`: The neighbor cell is empty or has no geometry.
  - `Occupied`: The neighbor cell contains geometry that may occlude.
  - `Unknown`: The neighbor state is not yet determined or accessible [S1].

**2D Coverage Mask**
A representation of the footprint of a polygon on a cell boundary [S1].
- Resolution: 1000x1000 integer grid per boundary face [S1].
- Representation: An exact union of axis-aligned 2D rectangles (AABBs) on this grid, derived by merging scanline intervals [S1].
- Note: While a dense bit mask can be used internally, the logical representation must be equivalent to the union of rectangles [S1].

### Steps

1. **Neighbor Query**: For the current cell at position $P$, query the six neighboring cells in directions $d \in \{+x, -x, +y, -y, +z, -z\}$ [S1].
   - Call `_outsideCell(P + offset(d))` to get the `OutsideCell` state [S1].
   - If the neighbor is `Empty` or `Unknown`, the boundary in direction $d$ is considered **open** [S1].

2. **Coverage Derivation (VXL-044)**: For each neighbor that is `Occupied`, derive its 2D coverage mask on the boundary plane opposite to the connection [S1].
   - Identify `outerShell` polygons of the neighbor that are coplanar to the boundary plane [S1].
   - **Rasterization**: Rasterize the edges of these coplanar polygons onto the 1000x1000 integer grid using a deterministic Bresenham-like integer rule [S1].
   - **Filling**: Fill the surface defined by the rasterized edges. Edges alone are insufficient; the filled surface is required [S1].
   - **Union**: Represent the covered cells as an exact union of axis-aligned 2D rectangles (AABBs) on this grid, e.g., by merging scanline intervals [S1]. This union must be equivalent to the discretized mask, including for triangles [S1].
   - **Orientation**: Apply orientation/flip transformations to align the 2D coordinates of the neighbor's mask with the current cell's boundary plane [S1].

3. **Closure Test**: Determine if a boundary direction $d$ is **closed** (occluded) [S1].
   - A direction is closed **only if** the neighbor's opaque `outerShell` coverage fully covers the entire 1000x1000 square of the boundary at the discrete resolution [S1].
   - The test uses **only** the neighbor's coverage, not a union with the current cell's `outerShell` [S1].
   - The neighbor's material must be fully opaque; non-totally opaque coverages (translucent, cutout) do not close the boundary [S1].
   - An AABB bounding box of a triangle or slope can only be used for early rejection, never as an exact coverage mask [S1].

4. **InnerShell Emission Decision (VXL-046)**: Check all six directions [S1].
   - If **all six** directions are closed (i.e., all six neighbors are `Occupied` and their opaque outerShell coverage fully covers the respective 1000x1000 boundary), then `emitInnerShell = false` [S1].
   - If **at least one** direction is open, partially covered, or unknown (i.e., neighbor is `Empty`, `Unknown`, or coverage is incomplete), then `emitInnerShell = true` [S1].
   - This rule applies even to Shapes with no `outerShell` polygons; they are not removed simply because their own outerShell coverages are empty [S1].

5. **Mesh Construction**:
   - Process `outerShell` visible fragments separately [S1].
   - If `emitInnerShell` is true, emit all `innerShell` polygons for the cell [S1].
   - If `emitInnerShell` is false, emit no `innerShell` polygons for the cell [S1].
   - Do not perform individual culling of inner polygons based on this rule; the decision is cell-wide [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_cell_mesh(volume, position):
    """
    Bakes the mesh for a single cell, determining inner shell emission.
    """
    # Step 1 & 2: Check neighbors and their coverage
    all_sides_closed = True
    
    for side in ALL_SIX_SIDES:
        # Query neighbor state
        neighbor = get_outside_cell(volume, position + offset(side))
        
        # If neighbor is not occupied (Empty/Unknown), boundary is open
        if neighbor.state != State.Occupied:
            all_sides_closed = False
            break
            
        # Derive 2D coverage mask from neighbor's outerShell
        # This involves rasterizing coplanar polygons onto 1000x1000 grid
        # and computing the union of AABBs
        coverage_mask = derive_2d_coverage_mask(neighbor.shape.outerShell, side)
        
        # Check if coverage fully covers the 1000x1000 boundary
        # Must account for orientation/flip of the neighbor
        if not is_fully_covered(coverage_mask, resolution=1000):
            all_sides_closed = False
            break
            
    # Step 4: Emission Decision
    emit_inner_shell = not all_sides_closed
    
    # Step 5: Construct Mesh
    mesh = Mesh()
    
    # Process outerShell visible fragments separately
    mesh.add_polygons(neighbor_cull_outer_shell(volume, position))
    
    # Emit innerShell based on decision
    if emit_inner_shell:
        mesh.add_polygons(get_inner_shell_polygons(position))
        
    return mesh

def derive_2d_coverage_mask(polygons, side):
    """
    Rasterizes coplanar polygons onto a 1000x1000 grid and returns
    the union of AABBs representing the covered area.
    """
    grid_size = 1000
    # Initialize empty coverage structure (e.g., list of AABBs or bit mask)
    covered_cells = set()
    
    for poly in polygons:
        if is_coplanar(poly, side):
            # Rasterize edges using deterministic Bresenham-like integer rule
            edges = rasterize_edges(poly, grid_size)
            # Fill surface
            filled = fill_surface(edges, grid_size)
            # Add to covered cells
            covered_cells.update(filled)
            
    # Convert to union of AABBs (e.g., by merging scanline intervals)
    aabb_union = convert_to_aabb_union(covered_cells, grid_size)
    return aabb_union

def is_fully_covered(coverage_mask, resolution):
    """
    Checks if the coverage mask covers the entire resolution x resolution square.
    """
    total_cells = resolution * resolution
    # If using AABB union, check if union covers all cells
    # If using bit mask, check if all bits are set
    return count_covered_cells(coverage_mask) == total_cells
```

### Parameters
- **Grid Resolution**: 1000x1000 integer grid per boundary face [S1].
- **Input Geometry Constraint**: Source polygons are planar and convex. Complex/concave shapes are assemblies of such polygons [S1].

### Edge Cases and Pitfalls
- **Empty/Unknown Neighbors**: Do NOT close a boundary. A missing neighbor or non-opaque material means the boundary is open [S1].
- **Partial Coverage**: Does NOT close a boundary. The test requires full coverage of the 1000x1000 square [S1].
- **AABB Bounding Box**: Can only be used for early rejection, never as an exact coverage mask [S1].
- **Interior Cameras**: This optimization is view-independent and may hide surfaces visible from inside a fully enclosed cavity. The behavior for interior cameras/cavities is an open issue not resolved by this spec [S1].
- **Degeneracy**: Diagonals and very small fragments are quantized. Degeneracies, holes, and potential rasterization fissures require a shared deterministic boundary contract [S1].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of polygons in the Shape's `outerShell`, per boundary face. Rasterization cost is constant per face (1000x1000 grid) [S1].
- **Memory Complexity**: $O(1)$ auxiliary per face for the union of rectangles, plus storage for the 6 masks if cached [S1].
- **Strengths**: Deterministic rasterization; avoids floating-point precision issues of analytical clipping; simplifies implementation [S1].
- **Limitations**: View-independent; may hide surfaces visible from inside a fully enclosed cavity; resolution-dependent (1/1000) [S1].
- **When to Use**: During synchronous mesh baking for static voxel models where deterministic occlusion is required [S1].
- **When Not to Use**: For per-frame GPU culling or dynamic scenes requiring view-dependent optimization [S1].

## Background
- **Origin**: VXL-044 and VXL-046 are part of the Sparkle voxel engine design specification (V0.1.4) by EreliaStudio [S1].
- **History**: VXL-044 replaces exact rational-coordinate geometric clipping with discrete 2D grid coverage to simplify implementation and ensure deterministic rasterization [S1]. VXL-046 validates the conditional innerShell emission rule based on six-neighbor coverage [S1].
- **Usage**: Used in the `VoxelMesher` bake phase of the Sparkle engine [S1].

## Sources
[S1] V0.1.4 — Voxel engine design and implementation specification, https://github.com/EreliaStudio/Sparkle/pull/37, fetched 2026-10-26
