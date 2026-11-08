# Watt & Watt Marching Cubes


## Overview
Watt & Watt Marching Cubes (WWMC) is an optimized implementation of the Marching Cubes algorithm that addresses four specific deficiencies in the original Lorensen and Cline method: topological inconsistencies on shared faces, performance overhead from processing non-active cells, redundant vertex calculations, and inefficient memory management for large datasets [S1]. It builds on [Lookup Table and Configuration](4978-lookup-table-and-configuration.md) by integrating a modified lookup table for topological correctness and a vertex deduplication scheme to reduce computational cost and output size [S1]. Unlike search-structure approaches that optimize for repeated queries, WWMC is designed for efficiency when the dataset is queried only once or a few times, using exhaustive cell coverage with buffering to handle data larger than main memory [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset of size $X_{max} \times Y_{max} \times Z_{max}$, stored as $Z_{max}$ slices, each containing $X_{max} \times Y_{max}$ values of $B$ bytes [S1].
*   **Output:** A triangular mesh represented by two tables:
    *   **Vertex Table:** Contains the 3D coordinates of unique vertices [S1].
    *   **Triangle Table:** Contains indices referencing positions in the Vertex Table, grouped in triplets to form triangles [S1].

**Data Structures:**
1.  **Modified Lookup Table:** A variant of the standard `triangleTable` and `edgeTable` from [Lookup Table and Configuration](4978-lookup-table-and-configuration.md) that resolves topological ambiguities (cases 46, 47, 98, 99, 210, 213, 214, 215) to ensure consistent triangulations on shared faces [S1]. The specific modification is referenced as [25] in the source [S1].
2.  **Vertex Index Map:** An auxiliary data structure (e.g., a hash map or array) that maps the geometric coordinates of a vertex to its index in the Vertex Table. This enables the deduplication of vertices shared between adjacent cells [S1].
3.  **Buffers:**
    *   **Input Buffer:** Holds $N_S$ slices ($N_S \le Z_{max}$) to process data in chunks [S1].
    *   **Output Buffers:** Temporary storage for vertices and triangles before writing to disk or final assembly [S1].

**Algorithm Steps:**

1.  **Buffer Initialization:** Allocate input and output buffers based on available RAM and dataset size. Load the first $N_S$ slices into the input buffer [S1].
2.  **Cell Classification:** Iterate through all cells in the current input slice pair. For each cell, determine if it is "active" by creating an 8-bit bit vector. For each of the 8 vertices $v_i$ ($i=0..7$) of the cell, set bit $i$ to 1 if $V(v_i) \ge I$ (isovalue), otherwise 0 [S1].
    *   A cell is **active** if the resulting bit vector value is not 0 (all vertices below isovalue) and not 255 (all vertices above or equal to isovalue) [S1].
    *   If inactive, skip to the next cell [S1].
3.  **Vertex Deduplication and Triangulation:** For each active cell:
    *   Determine the connectivity schema using the modified lookup table based on the 8-bit case index [S1].
    *   For each edge intersection required by the schema:
        *   Calculate the intersection point $P$ as a linear combination of the values at the intersected edge's endpoints [S1]. Specifically, if edge endpoints are $A$ and $B$ with values $V_A, V_B$ and isovalue $I$, the interpolation factor $t = (I - V_A) / (V_B - V_A)$, and $P = A + t(B - A)$ [G].
        *   Check if vertex $P$ already exists in the Vertex Table using the Vertex Index Map.
        *   If $P$ is new:
            *   Append $P$ to the Vertex Table.
            *   Store the new index in the Vertex Index Map.
            *   Add the index to the output vertex buffer.
        *   If $P$ exists, retrieve its existing index from the Vertex Index Map [S1].
    *   Form triangles using the indices retrieved/stored for the intersection points, as defined by the lookup table [S1].
    *   Add the triangle indices to the output triangle buffer [S1].
4.  **Buffer Management:** When the input buffer is exhausted, load the next $N_S$ slices. Continue until all $Z_{max}$ slices are processed.
5.  **Final Output:** Write the accumulated Vertex Table and Triangle Table to the output format (e.g., OFF or VRML) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def watt_watt_marching_cubes(volume, isovalue, x_max, y_max, z_max, slice_buffer_size):
    """
    volume: 3D array or file-like object supporting slice access
    isovalue: float
    x_max, y_max, z_max: int
    slice_buffer_size: int, number of Z-slices to process at once
    """
    vertex_table = []
    triangle_table = []
    vertex_index_map = {} # Maps (x,y,z) tuple to index in vertex_table
    
    # Load modified lookup tables from parent article
    edge_table, triangle_table_config = load_modified_lookup_tables()
    
    # Process volume in slices
    num_slices_to_process = (z_max - 1)
    for z_start in range(0, num_slices_to_process, slice_buffer_size):
        z_end = min(z_start + slice_buffer_size, num_slices_to_process)
        
        # Load input buffer
        input_slices = []
        for z in range(z_start, z_end + 1):
            input_slices.append(volume.get_slice(z))
            
        # Process cells in the buffer
        for z in range(z_start, z_end):
            slice_bottom = input_slices[z - z_start]
            slice_top = input_slices[z - z_start + 1]
            
            for y in range(y_max - 1):
                for x in range(x_max - 1):
                    # Get 8 vertices of the cell
                    # Vertices 0-3: bottom face (z), 4-7: top face (z+1)
                    # Ordering: 0=(x,y), 1=(x+1,y), 2=(x+1,y+1), 3=(x,y+1)
                    v = [
                        slice_bottom[y][x],
                        slice_bottom[y][x+1],
                        slice_bottom[y+1][x+1],
                        slice_bottom[y+1][x],
                        slice_top[y][x],
                        slice_top[y][x+1],
                        slice_top[y+1][x+1],
                        slice_top[y+1][x]
                    ]
                    
                    # Cell Classification
                    case_index = 0
                    for i in range(8):
                        if v[i] >= isovalue:
                            case_index |= (1 << i)
                            
                    # Skip inactive cells
                    if case_index == 0 or case_index == 255:
                        continue
                    
                    # Active Cell Triangulation
                    # Get triangle configuration from modified lookup table
                    # triangle_table_config[case_index] returns a list of edge indices
                    # terminated by -1
                    
                    tri_config = triangle_table_config[case_index]
                    edge_indices = []
                    for idx in tri_config:
                        if idx == -1:
                            break
                        edge_indices.append(idx)
                        
                    # Compute vertices for this cell
                    cell_vertices = []
                    for edge_idx in edge_indices:
                        # Determine edge endpoints based on standard edge numbering
                        # from parent article:
                        # e0=(0,1), e1=(1,2), e2=(2,3), e3=(3,0)
                        # e4=(4,5), e5=(5,6), e6=(6,7), e7=(7,4)
                        # e8=(0,4), e9=(1,5), e10=(2,6), e11=(3,7)
                        
                        v1_idx, v2_idx = get_edge_vertices(edge_idx)
                        p1 = v[v1_idx]
                        p2 = v[v2_idx]
                        
                        # Get geometric coordinates of the cell vertices
                        # Assuming unit cube spacing for interpolation
                        coords1 = get_vertex_coords(x, y, z, v1_idx, x_max, y_max)
                        coords2 = get_vertex_coords(x, y, z, v2_idx, x_max, y_max)
                        
                        # Linear interpolation
                        if p2 == p1:
                            t = 0.0
                        else:
                            t = (isovalue - p1) / (p2 - p1)
                            
                        intersection_point = lerp(coords1, coords2, t)
                        
                        # Vertex Deduplication
                        if intersection_point not in vertex_index_map:
                            idx = len(vertex_table)
                            vertex_table.append(intersection_point)
                            vertex_index_map[intersection_point] = idx
                            cell_vertices.append(idx)
                        else:
                            cell_vertices.append(vertex_index_map[intersection_point])
                            
                    # Add triangles
                    # The lookup table provides pairs of edge indices that form triangles
                    # We need to map the edge indices to the computed vertex indices
                    # The standard triangle table lists edge indices for each triangle
                    # We must re-fetch the specific triangle definitions for this case
                    
                    triangles_for_case = get_triangles_for_case(case_index)
                    for tri in triangles_for_case:
                        # tri is a tuple of 3 edge indices
                        v_indices = []
                        for e_idx in tri:
                            # Find the index in cell_vertices corresponding to e_idx
                            # This requires mapping edge_idx to position in edge_indices list
                            # Or simpler: re-calculate the vertex index for each edge in the triangle definition
                            # However, we already computed them in order. 
                            # Let's assume we stored them in a dict: edge_idx -> vertex_index
                            pass 
                            
                    # Simplified triangle generation using the pre-computed cell_vertices list
                    # which corresponds to the order of edges in edge_indices
                    # The lookup table gives edge indices. We need to find their position in edge_indices
                    # to get the correct vertex index.
                    
                    edge_to_vertex = {e: v for e, v in zip(edge_indices, cell_vertices)}
                    
                    # Get triangle definitions for this case_index from the modified table
                    # The table usually stores edge indices.
                    tri_defs = get_triangle_definitions(case_index)
                    for tri_def in tri_defs:
                        v0 = edge_to_vertex[tri_def[0]]
                        v1 = edge_to_vertex[tri_def[1]]
                        v2 = edge_to_vertex[tri_def[2]]
                        triangle_table.append((v0, v1, v2))

    return vertex_table, triangle_table

def get_edge_vertices(edge_idx):
    # Standard edge numbering from parent article
    edges = [
        (0, 1), (1, 2), (2, 3), (3, 0), # Bottom
        (4, 5), (5, 6), (6, 7), (7, 4), # Top
        (0, 4), (1, 5), (2, 6), (3, 7)  # Vertical
    ]
    return edges[edge_idx]

def get_vertex_coords(x, y, z, vertex_idx, x_max, y_max):
    # Map vertex index 0-7 to (x,y,z) coordinates
    # 0: (x, y, z)
    # 1: (x+1, y, z)
    # 2: (x+1, y+1, z)
    # 3: (x, y+1, z)
    # 4: (x, y, z+1)
    # 5: (x+1, y, z+1)
    # 6: (x+1, y+1, z+1)
    # 7: (x, y+1, z+1)
    
    dx = [0, 1, 1, 0, 0, 1, 1, 0]
    dy = [0, 0, 1, 1, 0, 0, 1, 1]
    dz = [0, 0, 0, 0, 1, 1, 1, 1]
    
    return (x + dx[vertex_idx], y + dy[vertex_idx], z + dz[vertex_idx])

def lerp(p1, p2, t):
    return (p1[0] + t*(p2[0]-p1[0]), p1[1] + t*(p2[1]-p1[1]), p1[2] + t*(p2[2]-p1[2]))
```

## Subtopics

### Parallel Marching Cubes (General Framework)
A parallelization strategy for WWMC that subdivides cells among several parallel processes, leveraging the independence of cell treatment [S1]. The data unit for partitioning is a pair of slices, corresponding to $(X_{max}-1) \times (Y_{max}-1)$ cells, which keeps I/O overheads low and permits fine-grain partitions [S1]. A key pitfall is the duplication of vertices lying on shared slices between consecutive intervals; these are "shared vertices," and triangles containing them are "shared triangles" [S1]. The goal of the spatial subdivision is to ensure each process receives a part of the volume with approximately the same number of triangles and low amounts of duplicated vertices [S1].

## Operating characteristics
**Time Complexity:**
The computational cost is defined as [S1]:
$$ T_{WWMC} = T_I(numC) + T_O(numAC) + K_{CC} \times numC + K_{ACT} \times numAC $$
where:
*   $numC$ is the total number of cells in the dataset.
*   $numAC$ is the number of active cells.
*   $T_I(numC)$ and $T_O(numAC)$ are I/O times for input and output, respectively [S1].
*   $K_{CC}$ is the constant time cost for Cell Classification per cell.
*   $K_{ACT}$ is the constant time cost for Active Cell Triangulation per active cell [S1].

**Measured Performance:**
On the workstation analyzed in the source, $K_{CC}$ is a constant value of $8.36 \times 10^{-8}$ seconds [S1]. $K_{ACT}$ varies between $3.1 \times 10^{-7}$ seconds and $5.7 \times 10^{-7}$ seconds, depending on the number of triangles produced by different kinds of active cells [S1]. The Active Cell Triangulation operation costs approximately one magnitude order more than the Cell Classification operation [S1].

**Scaling and Limitations:**
*   **Strengths:** WWMC is more efficient than search-structure approaches when the dataset is queried only once or a few times, as it avoids the overhead of building and accessing search structures [S1]. The vertex deduplication reduces the size of the Vertex table and computing time [S1]. Buffering allows processing of datasets larger than main memory [S1].
*   **Limitations:** Execution time is proportional to the size of the input dataset ($numC$), which can be high for large volumes [S1]. The algorithm requires careful management of shared vertices in parallel implementations to avoid topological inconsistencies [S1].

## Background
The Watt & Watt Marching Cubes (WWMC) algorithm was introduced by D. D'Agostino, A. Clematis, and V. Gianuzzi in 2000 [S1]. It is named after Watt & Watt because the first three issues addressed (topological consistency, performance, and vertex deduplication) were included in the version proposed in [27] (Watt & Watt, 1992), while the fourth issue (I/O buffering) was added by the authors [S1]. The algorithm addresses issues widely discussed in literature regarding the original Lorensen and Cline proposal [S1]. It is used in 3D data analysis workflows, particularly in medical image processing and scientific visualization, where efficient isosurface extraction from large volumetric datasets is required [S1].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
