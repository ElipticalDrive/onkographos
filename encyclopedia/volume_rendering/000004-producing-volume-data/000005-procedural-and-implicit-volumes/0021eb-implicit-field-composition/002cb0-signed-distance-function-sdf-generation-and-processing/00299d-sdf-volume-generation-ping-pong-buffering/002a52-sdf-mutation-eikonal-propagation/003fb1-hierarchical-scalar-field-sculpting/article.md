# Hierarchical Scalar Field Sculpting

Also called: Resolution Adaptive Volume Sculpting, ntree subdivision

## Overview
Hierarchical Scalar Field Sculpting is an interactive modeling technique that modifies a scalar field representing volumetric material density using user-defined tools [S1]. It solves the problem of maintaining interactivity during the creation of topologically complex, high-detail objects by adaptively refining the sampling resolution only where the tool influences the field [S1]. It sets itself apart from fixed-resolution sculpting by allowing local refinement without global re-meshing, and from delta-based multiresolution schemes by storing absolute field values at each level, ensuring vertical independence [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical scalar field representation (the `ntree`), a `Tool` (a scalar field function), and an `Action` (a combination rule).
*   **Output:** A modified scalar field with updated vertex values and potentially subdivided cells.

**Data Structures:**
1.  **Vertex:** Stores the scalar field value, field gradient, and color [S1].
2.  **ntree Cell:** A hierarchical structure where a Cell contains its sub-cells (children) rather than replacing them. The subdivision factor is $n=4$ per dimension [S1]. Cells share `Vertex` nodes on common faces or edges with adjacent cells at the same level [S1].
3.  **Vertex Cache:** An STL hash map indexed by memory addresses of `Vertex` objects. It maps the `Vertex` state prior to the current tool's modification. One cache is created per Tool-Action application pair [S1].

**Parameters:**
*   **Field Clamping Range:** $[0, 3]$ [S1].
*   **Iso-value:** $1.5$ [S1].
*   **Min Resolution:** The smallest cell size allowed, derived from the tool's minimum security cell size [S1].
*   **Max Resolution:** The largest cell size allowed (finest subdivision), derived from the tool's scale to prevent infinite loops [S1].
*   **Acceptable Discrepancy:** A threshold for `estimateDiscrepancy()` [S1].

**Algorithm Steps:**

1.  **Initialization:** Create a `Vertex Cache` for the current Tool-Action pair. Store the current state of all affected vertices in the cache [S1].
2.  **Apply Tool (Recursive):** For each `Cell` in the hierarchy:
    a.  Iterate over all `Vertex` nodes $v$ belonging to the cell.
    b.  Compute the tool's contribution $T(v)$ at the vertex location.
    c.  Apply the `Action` to update the vertex attributes. For an `AddAction`, this involves adding the tool's field value to the vertex field value, using these values as weights to sum attributes (gradient, color), and clamping the field value between 0 and 3 [S1].
    d.  If the cell has no children, call `checkSubdivide(t)` [S1].
    e.  If the cell has children, recursively call `apply(t, a)` on each child [S1].
3.  **Subdivision Check:** For a leaf `Cell` with size $s$ and tool $t$:
    a.  If $s > t.\text{getMinResolution}()$, subdivide the cell [S1].
    b.  If $s < t.\text{getMaxResolution}()$, calculate `estimateDiscrepancy()`. If the discrepancy exceeds `acceptableDiscrepancy`, subdivide the cell [S1].
4.  **Subdivision Process:**
    a.  Create sub-cells (children) with subdivision factor $n=4$ [S1].
    b.  Share `Vertex` nodes among common faces/edges of adjacent new cells [S1].
    c.  Initialize new vertices by interpolating field values from the parent cell's state *prior* to the current tool application (retrieved from the `Vertex Cache`) [S1]. This ensures the tool's modification is added correctly to the pre-existing field [S1].
5.  **Simplification (Idle Pass):** At idle moments, perform a simplification pass over the whole field. Destroy any `Cell` that does not cross the surface (iso-value 1.5) and whose children also do not cross the surface [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_tool_to_cell(cell: Cell, tool: Tool, action: Action, vertex_cache: dict):
    """
    Recursively applies a tool to a cell and its children.
    """
    # 1. Update current level vertices
    for vertex in cell.vertices:
        # Compute tool contribution at vertex location
        tool_value = tool.evaluate(vertex.position)
        
        # Apply action (e.g., AddAction)
        # For AddAction: new_field = clamp(old_field + tool_value, 0, 3)
        # Attributes are weighted sums based on old and new field values
        action.update(tool_value, vertex)
        
        # Store current state in cache for potential undo or interpolation
        # Note: Cache is keyed by memory address
        vertex_cache[id(vertex)] = vertex.clone()

    # 2. Handle children
    if not cell.has_children():
        # Leaf node: check if subdivision is needed
        if cell.size > tool.get_min_resolution():
            cell.subdivide()
        elif cell.size < tool.get_max_resolution():
            if cell.estimate_discrepancy() > cell.acceptable_discrepancy:
                cell.subdivide()
    else:
        # Internal node: recurse
        for child in cell.children:
            apply_tool_to_cell(child, tool, action, vertex_cache)

def subdivide_cell(cell: Cell, vertex_cache: dict):
    """
    Subdivides a cell and initializes new vertices from cached pre-update state.
    """
    # Create sub-cells with n=4 subdivision factor
    new_children = []
    for i in range(4):
        for j in range(4):
            for k in range(4):
                new_children.append(Cell())
    
    cell.children = new_children
    
    # Share vertices on common faces/edges
    # ... (implementation details of sharing logic) ...
    
    # Initialize new vertices
    for new_vertex in cell.get_new_vertices():
        # Interpolate from parent's pre-update state
        # The parent's vertices were cached before the tool application
        # We interpolate the cached values to get the base field value
        base_value = interpolate_from_cache(new_vertex.position, vertex_cache)
        new_vertex.field_value = base_value
        # Gradient and color also interpolated from cache
```

**Edge Cases and Pitfalls:**
*   **Clamping:** Field values must be clamped to $[0, 3]$ to prevent negative values (which block material creation) or excessive positive values (which block material removal) [S1].
*   **Infinite Subdivision:** Subtractive tools can create discontinuities. A `Max Resolution` (smallest cell size) is required to stop subdivision [S1].
*   **Reversibility:** Field clamping makes the operation non-reversible. A `Vertex Cache` is mandatory to store pre-update states for interpolation during subdivision and for undo operations [S1].
*   **Update Order:** Coarser levels must be updated first to provide interactive visual feedback. This requires careful initialization of new vertices from the cache [S1].

## Subtopics

### [Volume Sculpting](11209-volume-sculpting.md)
Volume sculpting is a free-form interactive modeling technique based on the metaphor of sculpting and painting a voxel-based solid material, such as a block of marble or wood. It solves the difficulty of modeling topologically complex and highly-detailed objects in traditional CAD systems and enables scientists and physicians to explore inner structures of volumetric datasets by gradually removing material. It sets itself apart from surface-based sculpting by operating directly on the volume buffer, allowing for localized rendering updates and Boolean operations between independent volume buffers.

## Operating characteristics
*   **Time Complexity:** Depends on the number of vertices affected by the tool and the depth of subdivision. Updating coarser levels first provides interactive feedback, but simplification is deferred to idle moments [S1].
*   **Memory Complexity:** Higher than fixed-resolution grids due to the hierarchical structure and vertex sharing overhead. The `ntree` structure does not allow classical octree storage optimizations because successors are enriched rather than replaced [S1].
*   **Strengths:** Interactive visual feedback, local refinement, smooth blending of tools (C1 variations), computationally cheap tool evaluation (squared distance) [S1].
*   **Limitations:** Simplification is drastic and deferred. Delta-based storage is not used, leading to potential redundancy. Undo mechanism relies on caching all modified vertices [S1].
*   **When to use:** Interactive modeling of complex organic shapes where local detail is needed.
*   **When not to use:** When global consistency or strict memory limits are critical, or when delta-based compression is preferred.

## Background
Hierarchical Scalar Field Sculpting was introduced by Eric Ferley, Marie-Paule Cani, and Jean-Dominique Gascuel in the paper "Resolution Adaptive Volume Sculpting" [S1]. It builds on Wyvill's field function for smooth blending and extends the concept of adaptive subdivision to volume sculpting. The method was developed to address the limitations of fixed-resolution voxel-based sculpting, allowing for interactive editing of high-detail models [S1].

## Sources
[S1] Resolution Adaptive Volume Sculpting, https://doi.org/10.1006/gmod.2001.0558, fetched 2026-10-26
[S2] Volume Sculpting, book, Not given in the sources.
