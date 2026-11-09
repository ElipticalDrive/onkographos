# EdgeMap Data Layout
Also called: EdgeMap

## Overview
The EdgeMap Data Layout is a dense grid structure that stores geometric intersection data (positions, normals, and traversal directions) for volume rendering isosurfaces. It solves the problem of efficiently storing per-edge intersection information required for marching cubes or similar voxel-to-mesh conversion algorithms. It differs from standard voxel grids by focusing on the edges of the voxel cells rather than the cell centers or vertices, providing a compact representation of the isosurface geometry [S1, unofficial].

## How it works
The EdgeMap Data Layout organizes data into a single `EdgeMap` struct containing four parallel arrays, all sized to `Index.TotalSize`. This size corresponds to the total number of edges in the 3D grid volume.

**Inputs and Outputs:**
- Input: A 3D volume grid of scalar values and an isovalue threshold.
- Output: An `EdgeMap` instance containing intersection data for all active edges.

**Data Structures:**
The `EdgeMap` struct contains the following fields [S1, unofficial]:
1. `Index`: An `IndexMap` object that likely manages the mapping of local edge indices to global indices or provides the `TotalSize`.
2. `Intersections`: A `unique_ptr<glm::vec3[]>` storing the 3D coordinates of the intersection point on each edge.
3. `Normals`: A `unique_ptr<glm::vec3[]>` storing the surface normal at each intersection point.
4. `Directions`: A `unique_ptr<float[]>` storing traversal direction information (likely for ray marching or connectivity).
5. `IndexOffset`: A `unique_ptr<int[]>` storing offset values, potentially for indexing into a vertex buffer or neighbor list.

All arrays are contiguous and sized to `Index.TotalSize`. The exact definition of `IndexMap` and how `TotalSize` is calculated from grid dimensions ($W \times H \times D$) is not explicitly detailed in the provided source, but it must account for the number of edges in a 3D grid. In a standard 3D grid of size $W \times H \times D$ voxels, the number of edges is typically $3 \times (W+1) \times (H+1) \times (D+1)$ minus shared edges, or more precisely, the sum of edges along X, Y, and Z axes. However, without the specific `IndexMap` implementation, the exact formula for `TotalSize` is Not given in the sources.

**Algorithm Steps:**
1. Initialize the `EdgeMap` struct with arrays of size `Index.TotalSize`.
2. Iterate through all edges in the 3D grid.
3. For each edge, determine if it intersects the isosurface.
4. If an intersection exists:
   a. Calculate the intersection point using linear interpolation between the scalar values at the edge's endpoints.
   b. Calculate the normal at the intersection point, typically by interpolating gradients from the adjacent voxels.
   c. Store the intersection point in `Intersections`.
   d. Store the normal in `Normals`.
   e. Store direction information in `Directions`.
   f. Store index offset in `IndexOffset`.
5. Return the populated `EdgeMap`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class EdgeMap:
    def __init__(self, index_map):
        self.Index = index_map
        self.TotalSize = index_map.TotalSize
        self.Intersections = [None] * self.TotalSize
        self.Normals = [None] * self.TotalSize
        self.Directions = [0.0] * self.TotalSize
        self.IndexOffset = [0] * self.TotalSize

    def compute_intersections(self, volume_grid, isovalue):
        # Iterate over all edges in the grid
        # The exact iteration logic depends on IndexMap implementation
        # [G] Assuming a standard edge enumeration based on grid dimensions
        for edge_id in range(self.TotalSize):
            # Get the two vertices defining this edge
            v1, v2 = self.Index.get_edge_vertices(edge_id)
            
            # Get scalar values at vertices
            s1 = volume_grid[v1]
            s2 = volume_grid[v2]
            
            # Check for intersection
            if (s1 - isovalue) * (s2 - isovalue) < 0:
                # Calculate intersection point
                t = (isovalue - s1) / (s2 - s1)
                intersection = v1 + t * (v2 - v1)
                
                # Calculate normal (simplified, assuming gradient interpolation)
                # [G] Normal calculation details are not provided in source
                normal = self.calculate_normal(v1, v2, volume_grid)
                
                # Store data
                self.Intersections[edge_id] = intersection
                self.Normals[edge_id] = normal
                self.Directions[edge_id] = self.calculate_direction(v1, v2)
                self.IndexOffset[edge_id] = edge_id # Placeholder for offset logic
```

**Parameters:**
- `isovalue`: The threshold value for isosurface extraction. Typical values depend on the data range.
- `volume_grid`: The input scalar field.

**Edge Cases:**
- Precision issues near the isovalue can cause missed intersections.
- Boundary conditions for edges on the volume surface must be handled.

## Subtopics

### [Edge ID Calculation](1570-edge-id-calculation.md)
Determines the unique global identifier for a vertex on a specific edge of a voxel cell. It maps local geometric primitives to a global vertex buffer index, ensuring consistent adjacency across voxel boundaries.

### [Enhanced Volume Data Set (EVDS)](11040-enhanced-volume-data-set-evds.md)
Augments standard voxel grids with Boolean edge labels to encode local connectivity. It separates segmentation from geometric modeling and enables efficient topological analysis. It builds on EdgeMap Data Layout by defining the specific data structure for storing edge states.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of edges in the grid, $O(E)$, where $E$ is the total number of edges.
- **Memory Complexity:** Linear with respect to the number of edges, $O(E)$, storing three vectors and two integer arrays per edge.
- **Strengths:** Compact representation of intersection data; efficient for marching cubes algorithms.
- **Limitations:** Requires significant memory for large volumes; depends on accurate edge enumeration.
- **When to use:** When performing isosurface extraction from dense voxel data.
- **When not to use:** For sparse data or when memory is extremely constrained.

## Background
The EdgeMap Data Layout is part of the Isosurface project by HeartofTheForce. It is used in volume rendering applications to facilitate the conversion of voxel data into triangle meshes.

## Sources
[S1] HeartofTheForce/Isosurface, https://github.com/HeartofTheForce/Isosurface, fetched 2026-10-26 (unofficial)
