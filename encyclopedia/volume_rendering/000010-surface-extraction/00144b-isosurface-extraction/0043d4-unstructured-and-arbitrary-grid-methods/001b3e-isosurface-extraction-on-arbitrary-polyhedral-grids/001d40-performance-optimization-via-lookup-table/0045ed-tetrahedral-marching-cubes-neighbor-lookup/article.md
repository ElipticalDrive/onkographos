# Tetrahedral Marching Cubes Neighbor Lookup

Also called: Vertex Neighbor Lookup Table, Vertex neighbor lookup table around cube corner

## Overview
The Tetrahedral Marching Cubes Neighbor Lookup is a data structure used to efficiently map tetrahedra during isosurface extraction on unstructured or arbitrary polyhedral grids. It provides anchor displacements and neighbor tetrahedron offsets for vertices surrounding a specific cube corner, enabling the algorithm to locate relevant tetrahedra without exhaustive search. It is a performance optimization component of the Tetrahedral Marching Cubes method, specifically designed to handle the geometric complexity of decomposing a cube into tetrahedra.

## How it works
The method relies on a precomputed lookup table that maps vertex indices to their corresponding tetrahedron offsets and spatial displacements relative to a reference cube corner. This allows the isosurface extraction algorithm to quickly identify which tetrahedra share a vertex or are adjacent to it, facilitating the construction of the isosurface mesh.

### Inputs and Outputs
- **Input**: A cube corner index $c = (c_x, c_y, c_z) \in \{0, 1\}^3$.
- **Output**: A set of entries, each containing an index range $i$, an anchor displacement $\Delta_i = (\Delta_{x,i}, \Delta_{y,i}, \Delta_{z,i})$, and a neighbor tetrahedron offset $t_i$.

### Data Structures
The core data structure is a lookup table (referred to as Table A8 in the source). The table is organized by rows, where each row corresponds to a specific configuration of neighbor indices.

**Table Layout:**
- **Columns**:
  - $i$: The neighbor index (or range of indices).
  - $\Delta_x$: The x-component of the anchor displacement.
  - $\Delta_y$: The y-component of the anchor displacement.
  - $\Delta_z$: The z-component of the anchor displacement.
  - $t_i$: The offset to the neighbor tetrahedron.

**Example Rows for Corner $c=(0,0,0)$:**
- Indices $i=1–6$: $\Delta=(-1,-1,-1)$, $t_i=-5$
- Indices $i=7–8$: $\Delta=(-1,-1,0)$, $t_i=1,2$
- Indices $i=9–10$: $\Delta=(-1,0,-1)$, $t_i=0,5$
- Indices $i=11–12$: $\Delta=(-1,0,0)$, $t_i=0,1$
- Indices $i=13–14$: $\Delta=(0,-1,-1)$, $t_i=3,4$
- Indices $i=15–16$: $\Delta=(0,-1,0)$, $t_i=2,3$
- Indices $i=17–18$: $\Delta=(0,0,-1)$, $t_i=4,5$
- Indices $i=19–24$: $\Delta=(0,0,0)$, $t_i=0–5$

### Steps
1. **Select Cube Corner**: Identify the cube corner $c = (c_x, c_y, c_z)$ for which the neighbor lookup is required. The table is defined per cube corner because the vertices of each tetrahedron can be mapped to a specific cube corner.
2. **Retrieve Base Entries**: Look up the base entries in the lookup table corresponding to the configuration for corner $c$. If the table is stored for a canonical corner (e.g., $(0,0,0)$), retrieve those entries.
3. **Adjust Displacements**: For each entry in the table, adjust the anchor displacement $\Delta_i$ to account for the specific cube corner $c$. The adjustment is performed by adding the corner coordinates to the displacement components:
   $$ \Delta_i \leftarrow \Delta_i + c $$
   where $c = (c_x, c_y, c_z)$.
4. **Map Tetrahedra**: Use the adjusted displacements $\Delta_i$ and the neighbor tetrahedron offsets $t_i$ to map the tetrahedra associated with the vertices surrounding the cube corner. These configurations are used in the isosurface extraction process to identify the relevant tetrahedra for mesh generation.

### Parameters
- **Cube Corner $c$**: A 3D vector with components in $\{0, 1\}$. The table must be defined or adapted for each of the 8 possible cube corners.

### Edge Cases and Pitfalls
- **Self Configuration Coincidence**: For a specific cube corner $c$ and tetrahedron offset $t$, one row in the lookup table may coincide with the self configuration, where $\Delta_0 = (0, 0, 0)$ and $t_0 = t$. This must be handled carefully to avoid incorrect self-references or infinite loops in the neighbor traversal.
- **Precision and Boundaries**: The displacements $\Delta_i$ are integer offsets. Care must be taken when applying these offsets to ensure that the resulting tetrahedron indices remain within the bounds of the grid. Out-of-bounds checks are necessary.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_neighbor_lookup(c):
    """
    Retrieve and adjust the neighbor lookup table for a given cube corner c.
    
    Args:
        c (tuple): The cube corner coordinates (cx, cy, cz), each in {0, 1}.
        
    Returns:
        list: A list of tuples, each containing (i, delta_x, delta_y, delta_z, t_i).
    """
    # Base lookup table for corner (0, 0, 0)
    # Format: (start_index, end_index, delta_x, delta_y, delta_z, t_i_values)
    # Note: t_i_values can be a single value or a list/range
    base_table = [
        (1, 6, -1, -1, -1, -5),
        (7, 8, -1, -1, 0, [1, 2]),
        (9, 10, -1, 0, -1, [0, 5]),
        (11, 12, -1, 0, 0, [0, 1]),
        (13, 14, 0, -1, -1, [3, 4]),
        (15, 16, 0, -1, 0, [2, 3]),
        (17, 18, 0, 0, -1, [4, 5]),
        (19, 24, 0, 0, 0, list(range(0, 6)))
    ]
    
    adjusted_entries = []
    
    for start_i, end_i, dx, dy, dz, t_vals in base_table:
        # Adjust displacements by adding the cube corner coordinates
        adj_dx = dx + c[0]
        adj_dy = dy + c[1]
        adj_dz = dz + c[2]
        
        # Handle t_vals which might be a single value or a list/range
        if isinstance(t_vals, list):
            for t in t_vals:
                adjusted_entries.append((start_i, end_i, adj_dx, adj_dy, adj_dz, t))
        else:
            adjusted_entries.append((start_i, end_i, adj_dx, adj_dy, adj_dz, t_vals))
            
    return adjusted_entries
```

## Subtopics

### Tetrahedral Marching Cubes (5-tetrahedra method)
The parent method that uses this lookup table for efficient neighbor identification during isosurface extraction. It decomposes a cube into 5 tetrahedra and uses the lookup table to quickly find adjacent tetrahedra. [Tetrahedral Marching Cubes (5-tetrahedra method)](4547-tetrahedral-marching-cubes-5-tetrahedra-method.md)

## Operating characteristics
- **Time Complexity**: The lookup operation is $O(1)$ per cube corner, as it involves a simple table lookup and addition. The overall complexity of the isosurface extraction depends on the number of cubes and tetrahedra processed.
- **Memory Complexity**: The lookup table is small and fixed-size, requiring minimal memory. The size is proportional to the number of unique configurations per cube corner, which is constant.
- **Strengths**: Significantly reduces the computational cost of neighbor identification, leading to faster isosurface extraction. The precomputed nature of the table avoids redundant calculations.
- **Limitations**: The method is specific to the tetrahedral decomposition used. Changes to the decomposition or grid structure may require updating the lookup table.
- **When to Use**: Use this method when performing isosurface extraction on tetrahedral grids where efficient neighbor lookup is critical for performance.
- **When Not to Use**: Avoid if the grid structure is highly dynamic or if the tetrahedral decomposition changes frequently, as the lookup table would need constant updates.

## Background
The Tetrahedral Marching Cubes Neighbor Lookup was introduced in the context of analytic isosurface extraction on multi-resolution tetrahedral grids. It addresses the challenge of efficiently navigating the tetrahedral mesh during surface extraction, which is a common bottleneck in volume rendering and scientific visualization. The method is part of the TetraSDF framework, which aims to provide high-quality isosurfaces with minimal artifacts.

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
