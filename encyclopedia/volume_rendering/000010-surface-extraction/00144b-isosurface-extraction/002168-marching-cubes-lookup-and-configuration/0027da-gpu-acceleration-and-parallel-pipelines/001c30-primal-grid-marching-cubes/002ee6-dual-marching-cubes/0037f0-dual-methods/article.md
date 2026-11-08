# Dual Methods

Also called: Dual schemes, dual isosurface extraction

## Overview
Dual methods are a class of isosurface extraction algorithms that generate surface topology by creating an $(n-1)$-cell for every edge crossing the boundary in 3D, resulting in a mesh where topology is completely determined by the sample complex [S1, unofficial]. Unlike primal methods such as Marching Cubes, which suffer from topological ambiguity in non-simplicial cells, dual methods resolve ambiguities by construction, though they may produce non-manifold vertices [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: A volumetric mesh (sample complex) defined by scalar field values at grid nodes.
- Output: A dual isosurface mesh.

**Data structures:**
The method constructs a mesh where vertices correspond to $n$-cells (voxels in 3D) and faces correspond to boundary-crossing edges. No explicit lookup tables are required; the topology is derived directly from the sample complex structure.

**Steps:**
1. Identify all $n$-cells (e.g., voxels) intersected by the isosurface.
2. For every $n$-cell intersecting the boundary, create a vertex in the dual mesh. The position of this vertex is assigned based on a specific rule (e.g., the cell center for Surface Nets) [S1, unofficial].
3. For every $d$-dimensional cell in the sample complex that crosses the boundary, create an $(n-d)$-cell in the dual mesh [S1, unofficial].
   - Specifically, for every edge crossing the boundary, create a face (in 3D).
   - For every face crossing the boundary, create an edge (in 3D).
4. Connect the vertices and cells according to the dual incidence relations defined by the sample complex.

**Parameters:**
- Vertex assignment rule: Determines the position of the dual vertex within the $n$-cell. Common choices include the cell center or a weighted average of sample values. Not given in the sources.

**Edge cases and pitfalls:**
- Non-manifold vertices: The construction may result in non-manifold geometry, which can be problematic for certain rendering or simulation pipelines [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_extract(isosurface_value, grid_values, grid_topology):
    """
    Extracts an isosurface using a dual method.
    
    Args:
        isosurface_value: The scalar threshold for the isosurface.
        grid_values: 3D array of scalar values at grid nodes.
        grid_topology: Structure defining the connectivity of the sample complex (cells, edges, faces).
        
    Returns:
        dual_mesh: A mesh object containing vertices and faces.
    """
    dual_vertices = []
    dual_faces = []
    
    # Step 1 & 2: Create a vertex for every n-cell (voxel) intersecting the boundary
    for cell in grid_topology.cells:
        if cell_intersects_boundary(cell, grid_values, isosurface_value):
            # Assign vertex position based on a rule (e.g., cell center)
            # This is the "tricky part" of dual methods [S1, unofficial]
            vertex_position = assign_vertex_position(cell, grid_values, isosurface_value)
            dual_vertices.append(vertex_position)
            
    # Step 3: Create cells based on boundary-crossing lower-dimensional elements
    # For every edge crossing the boundary, create a face (in 3D)
    for edge in grid_topology.edges:
        if edge_intersects_boundary(edge, grid_values, isosurface_value):
            # Identify the adjacent cells (n-cells) connected to this edge
            adjacent_cells = get_adjacent_cells(edge, grid_topology)
            
            # Create a face connecting the dual vertices of these adjacent cells
            # The exact connectivity depends on the specific dual scheme
            face_vertices = [dual_vertices[cell.id] for cell in adjacent_cells]
            dual_faces.append(face_vertices)
            
    # Note: For every face crossing the boundary, create an edge (in 3D)
    # This step is implied by the general rule but often results in the same connectivity
    # as the face creation in standard dual schemes like Surface Nets.
    
    return Mesh(dual_vertices, dual_faces)

def cell_intersects_boundary(cell, grid_values, isosurface_value):
    """Check if a cell intersects the isosurface."""
    # Implementation depends on the specific grid and value distribution
    # Typically involves checking if the isosurface value lies between
    # the min and max values of the cell's vertices
    min_val = min(grid_values[v] for v in cell.vertices)
    max_val = max(grid_values[v] for v in cell.vertices)
    return min_val <= isosurface_value <= max_val

def edge_intersects_boundary(edge, grid_values, isosurface_value):
    """Check if an edge intersects the isosurface."""
    v1_val = grid_values[edge.v1]
    v2_val = grid_values[edge.v2]
    return (v1_val - isosurface_value) * (v2_val - isosurface_value) < 0

def assign_vertex_position(cell, grid_values, isosurface_value):
    """
    Assigns a position to the dual vertex within the cell.
    Common choice: Cell center [S1, unofficial].
    More complex choices may use weighted averages based on grid_values.
    """
    # Placeholder for specific vertex assignment rule
    # For Surface Nets, this is typically the cell center
    return cell.center
```

## Subtopics

### [Dual Mesh Building](1423-dual-mesh-building.md)
Focuses on the construction of the dual mesh structure itself.

### [Parallel Dual Marching Cube](2131-parallel-dual-marching-cube.md)
Optimizes dual methods for parallel execution on GPUs.

### [Dual Marching Cubes Vertex Processing](3688-dual-marching-cubes-vertex-processing.md)
Details the processing of vertices in dual methods.

## Operating characteristics
- **Time complexity:** Not given in the sources.
- **Memory complexity:** Not given in the sources.
- **Strengths:** Topology is completely determined by the sample complex, eliminating ambiguities found in primal methods [S1, unofficial].
- **Limitations:** May produce non-manifold vertices [S1, unofficial].
- **When to use:** When topological correctness is critical and non-manifold geometry is acceptable or can be handled.
- **When not to use:** When manifold geometry is strictly required.

## Background
The first dual method, Surface Nets, was proposed by Sarah Frisken Gibson in 1999 [S1, unofficial]. Research into dual schemes has continued since the 1980s, with a focus on exploring different rules for assigning vertices to $n$-cells [S1, unofficial].

## Sources
[S1] Smooth Voxel Terrain (Part 2) – 0 FPS, https://0fps.net/2012/07/12/smooth-voxel-terrain-part-2/, fetched 2026-10-28 (unofficial)
