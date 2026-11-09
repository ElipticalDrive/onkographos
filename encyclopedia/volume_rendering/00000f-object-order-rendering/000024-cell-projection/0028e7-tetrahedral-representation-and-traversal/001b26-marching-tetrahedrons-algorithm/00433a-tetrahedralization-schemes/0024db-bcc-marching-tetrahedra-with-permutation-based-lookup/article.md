# BCC Marching Tetrahedra with Permutation-Based Lookup

Also called: BCC MT with Reference Tetrahedron, GPU BCC Isosurface Extraction

## Overview
This method extracts explicit, connected triangle mesh isosurfaces from Body-Centered Cubic (BCC) volume datasets on a GPU [S1]. It solves the problem of large lookup tables required for tetrahedral decomposition by storing connectivity only for a single "Reference Tetrahedron" and applying 3D permutation transformations to handle all six cell types [S1]. Unlike naive approaches that require massive lookup tables for every cell variant, this method leverages the regular pattern of BCC lattice cells to reduce memory overhead for table storage, while producing a connected mesh structure that supports efficient post-processing like curvature rendering [S1].

## How it works
The method processes a BCC volume dataset to generate a triangle mesh. It relies on the fact that a BCC cell can be decomposed into tetrahedra, and while there are six distinct cell types based on vertex configurations, they share a topological structure related by permutation [S1].

### Inputs and Outputs
*   **Input:** A BCC volume dataset (scalar field values at lattice points), an isosurface threshold $T$, and a pre-allocated fixed-size GPU memory buffer for the output mesh [S1].
*   **Output:** A vertex buffer containing intersection points and an index buffer defining the triangles [S1].

### Data Structures
1.  **Reference Tetrahedron Lookup Tables:**
    *   **Edge Table (Table 2a in [S1]):** Maps a 5-bit key (representing the intersection pattern on an edge) to an edge direction index [S1].
    *   **Connectivity Table (Table 3 in [S1]):** Defines which edges form triangles for the Reference Tetrahedron based on the intersection pattern [S1].
    *   **Reference Edge Definition (Table 4 in [S1]):** Defines the 6 edges of the reference tetrahedron using lattice offsets and directions.
2.  **Buffers:**
    *   **Intersection Buffer:** Stores intermediate intersection data (5-bit key) generated during the marking phase [S1].
    *   **Vertex Buffer:** Stores the 3D coordinates of intersection points [S1].
    *   **Index Buffer:** Stores indices into the Vertex Buffer to form triangles [S1].
    *   **Constant Memory:** The Reference Lookup Tables are stored in GPU Constant Memory for fast access [S1].

### Reference Tetrahedron Edges
The reference tetrahedron has 6 edges defined by lattice offsets and directions [S1]:

| Edge | Lattice Offset | Direction |
| :--- | :--- | :--- |
| e0 | $(0,0,0)$ | $(-1, 1, 1)$ |
| e1 | $(-1,1,1)$ | $(1,-1, 1)$ |
| e2 | $(0,0,2)$ | $(1, 1,-1)$ |
| e3 | $(0,0,0)$ | $(1, 1, 1)$ |
| e4 | $(0,0,0)$ | $(0, 0, 2)$ |
| e6 | $(-1,1,1)$ | $(2, 0, 0)$ |

*Note: The source lists 'e6' but implies 6 edges total (e0-e5, with e5 possibly omitted or e6 being a typo for e5 in the source text, but we follow the source's explicit list) [S1].*

### Algorithm Steps

1.  **Marking Phase (Intersection Detection):**
    *   For each lattice point in the BCC grid, determine the cell type and identify edges intersecting the isosurface.
    *   Store the intersection information (5-bit key) in the Intersection Buffer [S1].
    *   *Note: The specific marking algorithm details are not fully provided in the source, but it precedes the vertex generation [S1].*

2.  **Vertex Buffer Generation:**
    *   Create one thread per intersection point (corresponding to an entry in the Intersection Buffer) [S1].
    *   Each thread reads the 5-bit key from the Intersection Buffer.
    *   Use the key to look up the edge direction from the Edge Table (Table 2a) [S1].
    *   Determine the current lattice point coordinates $V_{curr}$ and the edge direction $D$ [S1].
    *   Calculate the opposite lattice point coordinates: $V_{opp} = V_{curr} + D$ [S1].
    *   Read the scalar volume data values $S_1$ at $V_{curr}$ and $S_2$ at $V_{opp}$ [S1].
    *   Calculate the intersection position $P$ using linear interpolation:
        $$ P = V_{curr} + \frac{T - S_1}{S_2 - S_1} (V_{opp} - V_{curr}) $$
        where $T$ is the isosurface threshold [S1].
    *   Store $P$ in the Vertex Buffer [S1].

3.  **Triangle Buffer (Index) Generation:**
    *   Create one thread per edge/cell that requires triangle generation [S1].
    *   Determine the type of the edge/cell being processed. This involves identifying which of the 6 BCC cell types the current tetrahedron belongs to [S1].
    *   **Permutation Transformation:**
        *   Since only the Reference Tetrahedron's connectivity is stored, the thread must apply a permutation transformation to map the reference connectivity to the current cell type [S1].
        *   The 6 cell types connected to a lattice point are related by 3D permutation transformations [S1].
        *   Apply the specific permutation for the current cell type to the lattice offsets and edge directions stored in the Reference Lookup Tables (Table 3 and Table 4) [S1].
        *   In the GPU implementation, this permutation is performed using the OpenCL `shuffle` function on the data stored in Constant Memory [S1].
    *   **Triangle Assembly:**
        *   Use the permuted connectivity information to determine which edges form triangles [S1].
        *   Retrieve the indices of the vertices (from the Vertex Buffer) corresponding to the intersection points on these edges.
        *   Write the triangle indices to the Index Buffer [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
NUM_EDGES_REF = 6
# Reference Edge Definitions from Table 4
REF_EDGES = [
    {"offset": (0,0,0), "dir": (-1, 1, 1)}, # e0
    {"offset": (-1,1,1), "dir": (1,-1, 1)}, # e1
    {"offset": (0,0,2), "dir": (1, 1,-1)}, # e2
    {"offset": (0,0,0), "dir": (1, 1, 1)}, # e3
    {"offset": (0,0,0), "dir": (0, 0, 2)}, # e4
    {"offset": (-1,1,1), "dir": (2, 0, 0)}  # e6 (Source lists e6, assuming 6 edges)
]

# Lookup Tables (Stored in Constant Memory)
# Table 2a: Maps 5-bit key to edge direction index
EDGE_LOOKUP = [...] 
# Table 3: Connectivity for Reference Tetrahedron
# Maps intersection pattern to list of edge indices forming triangles
CONNECTIVITY_LOOKUP = [...]

def get_permutation(cell_type):
    """
    Returns the permutation transformation for the given BCC cell type.
    The 6 cell types are related by 3D permutations.
    Implementation detail: Uses OpenCL shuffle function in GPU.
    Here we simulate the permutation logic.
    """
    # Placeholder for specific permutation logic derived from BCC symmetry
    # The source states permutations are applied to lattice offsets and edge directions
    pass

def vertex_generation_thread(intersection_entry, volume_data, threshold):
    """
    Thread function for generating vertex positions.
    """
    key = intersection_entry.key
    lattice_point = intersection_entry.lattice_point
    
    # 1. Get edge direction from lookup table
    edge_dir_index = EDGE_LOOKUP[key]
    
    # 2. Get the reference edge definition
    ref_edge = REF_EDGES[edge_dir_index]
    
    # 3. Calculate opposite lattice point
    # Note: The source says "Add the edge direction to the current lattice point coordinates"
    # However, the edge direction in Table 4 is relative. 
    # We assume the intersection entry provides the base lattice point for the edge.
    opp_lattice_point = lattice_point + ref_edge["dir"]
    
    # 4. Read volume data
    s1 = volume_data[lattice_point]
    s2 = volume_data[opp_lattice_point]
    
    # 5. Linear Interpolation
    # Guard against division by zero
    if s2 == s1:
        return None # Degenerate case
    
    t = threshold
    p = lattice_point + ((t - s1) / (s2 - s1)) * (opp_lattice_point - lattice_point)
    
    return p

def triangle_generation_thread(cell_info, volume_data, threshold, vertex_buffer):
    """
    Thread function for generating triangle indices.
    """
    cell_type = cell_info.type
    lattice_point = cell_info.lattice_point
    
    # 1. Get permutation for this cell type
    permutation = get_permutation(cell_type)
    
    # 2. Apply permutation to Reference Connectivity
    # The connectivity table gives edge indices for the reference tetrahedron.
    # We need to map these to the actual edges of the current cell.
    # This involves permuting the lattice offsets and edge directions.
    
    permuted_connectivity = []
    for edge_indices in CONNECTIVITY_LOOKUP[cell_info.intersection_pattern]:
        permuted_edges = []
        for edge_idx in edge_indices:
            ref_edge = REF_EDGES[edge_idx]
            # Apply permutation to offset and direction
            # Note: The exact permutation application logic is complex and depends on BCC symmetry
            # The source mentions using OpenCL shuffle for this
            permuted_offset = apply_permutation(ref_edge["offset"], permutation)
            permuted_dir = apply_permutation(ref_edge["dir"], permutation)
            
            # Calculate the actual edge in the current cell
            # The edge connects two lattice points
            start_point = lattice_point + permuted_offset
            end_point = start_point + permuted_dir
            
            # Find the intersection points on this edge
            # This requires re-running the interpolation logic or looking up pre-computed vertex indices
            # The source implies the vertex buffer is already populated by the vertex generation phase
            # We need to find the index of the vertex corresponding to the intersection on this edge
            
            # Simplified: Assume we can find the vertex index based on the edge and lattice point
            # In practice, this requires a mapping from (lattice_point, edge) to vertex index
            v_idx_start = find_vertex_index(start_point, end_point, volume_data, threshold)
            v_idx_end = find_vertex_index(end_point, start_point, volume_data, threshold) # Direction might be reversed
            
            if v_idx_start is not None and v_idx_end is not None:
                permuted_edges.append((v_idx_start, v_idx_end))
        
        permuted_connectivity.append(permuted_edges)
    
    # 3. Write indices to Index Buffer
    for triangle_edges in permuted_connectivity:
        # A triangle is formed by 3 edges? Or the connectivity table gives vertices directly?
        # The source says "Connectivity Table... defines which edges form triangles"
        # Typically, MT outputs triangles defined by 3 vertices.
        # If the table gives edges, we need to connect them.
        # Assuming the connectivity table provides the vertices directly for the reference tetrahedron
        # and the permutation maps these vertices to the current cell's vertices.
        
        # Re-reading source: "Connectivity structure for the Reference Tetrahedron... apply permutation... to lattice offsets and edge directions"
        # This suggests the connectivity is defined by edges, and we must resolve vertices.
        
        # For the purpose of this pseudocode, we assume the connectivity table provides
        # a list of vertex indices for the reference tetrahedron, and we permute the lattice points
        # to find the corresponding vertices in the current cell.
        
        # This part is ambiguous in the source. We will assume a standard MT output:
        # The connectivity table provides a list of edge indices that form a triangle.
        # We need to resolve these edges to vertices.
        
        # Let's assume the connectivity table gives the vertices directly for simplicity,
        # as is common in MT implementations, and the "permutation" applies to the lattice points
        # to find the correct vertices in the vertex buffer.
        
        # However, the source explicitly mentions permuting "lattice offsets and edge directions".
        # This implies the connectivity is edge-based.
        
        # Given the ambiguity, we will state that the triangle indices are generated by
        # resolving the permuted edges to vertex indices in the Vertex Buffer.
        pass

def apply_permutation(vector, permutation):
    """
    Applies a 3D permutation to a vector.
    """
    # Example: permutation = (1, 0, 2) means x' = y, y' = x, z' = z
    # This is a placeholder for the actual BCC symmetry permutation logic
    return vector
```

### Parameters
*   **Isosurface Threshold ($T$):** The scalar value at which the isosurface is extracted.
*   **Reference Tetrahedron:** The base tetrahedron used for lookup. All other cell types are derived from it via permutation [S1].

### Edge Cases and Pitfalls
*   **Dynamic Mesh Size:** The size of the extracted triangle mesh is dynamic and depends on the isosurface shape. Pre-calculating the exact size is difficult. For iterative processes, pre-allocating a sufficiently large fixed memory space is more efficient than reallocating every frame [S1].
*   **Space Skipping Ineffectiveness:** The method relies on space skipping (culling empty cells). This is ineffective for datasets with high noise or complex isosurfaces with few empty spaces [S1].
*   **Memory Usage:** The method is more complex and requires more GPU memory than existing methods due to the need for vertex and index buffers and the lookup tables [S1].
*   **Degenerate Edges:** Linear interpolation requires $S_1 \neq S_2$. If $S_1 = S_2$, the edge does not cross the isosurface (or is tangent), and no intersection point should be generated.

## Subtopics

### [Optimized FCC Data Fetching](3418-optimized-fcc-data-fetching.md)
Optimized FCC Data Fetching is a branchless, lookup-table-free algorithm for determining the tetrahedral polynomial type and fetching the 19 required data samples for Box-Spline isosurface raycasting on Face-Centered Cubic (FCC) lattices. It solves the performance bottleneck of conditional branches and memory lookups inherent in naive tetrahedral decomposition methods. Unlike its sibling [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md), which handles BCC lattices via permutation tables, this method operates directly on FCC datasets by deriving a rotation vector and offset to align the local tetrahedral coordinate system, enabling a sequential, cache-friendly fetch pattern.

### [BCC GPU Marching Tetrahedra with Min/Max Block Skipping](5418-bcc-gpu-marching-tetrahedra-with-min-max-block-skipping.md)
This method extracts explicit, connected triangle mesh isosurfaces from Body-Centered Cubic (BCC) volume datasets on a GPU. It improves upon classical Marching Tetrahedra (MT) by using min/max blocks for empty space skipping, avoiding the "triangle soup" format and expensive post-processing. It builds on [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md) by adding a hierarchical culling stage to boost performance on high-resolution data.

## Operating characteristics
*   **Time Complexity:** The method's performance scales well with resolution. At low resolutions, it is >2x slower than classic Marching Tetrahedra (MT) due to overhead from creating the triangle mesh structure and low space skipping rates [S1]. At high resolutions, it is significantly faster (up to ~5x) due to high space skipping rates [S1]. The time increase with resolution is small [S1].
*   **Memory Complexity:** The method requires more GPU memory than existing methods due to the need for vertex and index buffers and the lookup tables [S1]. The dynamic mesh size makes pre-allocation necessary, which can lead to over-allocation if the isosurface is simple [S1].
*   **Measured Performance:**
    *   **Hardware:** Intel Core i5 2500 CPU @ 3.30GHz and NVIDIA GeForce RTX 2080 Ti GPU [S1].
    *   **Datasets:** Genus-3 (G1-G5) and Three-Tori (T1-T5) with resolutions ranging from $128^3$ to $1024 \times 512 \times 512$ (Genus-3) and $128^3$ to $512^3$ (Three-Tori) [S1].
    *   **Results:** At high resolutions (e.g., G5, T5), the method shows up to ~5x speedup over classic MT [S1].
*   **Strengths:**
    *   Produces a connected triangle mesh, which is more efficient for rendering and post-processing (e.g., curvature estimation) than triangle soup [S1].
    *   High performance on high-resolution datasets with significant empty space [S1].
    *   Reduced lookup table size compared to naive BCC MT implementations [S1].
*   **Limitations:**
    *   Higher memory usage than triangle soup methods [S1].
    *   Performance degrades on datasets with high noise or complex isosurfaces with few empty spaces, as space skipping becomes ineffective [S1].
    *   More complex implementation than classic MT [S1].
*   **When to Use:** High-resolution BCC volume datasets with significant empty space, where a connected mesh is required for further processing (e.g., curvature rendering) [S1].
*   **When Not to Use:** Datasets with high noise, complex isosurfaces with few empty spaces, or when memory is severely constrained [S1].

## Background
The method was proposed in a paper by Korean researchers, published in 2020 [S1]. It addresses the limitations of classic Marching Tetrahedra, which produces a "triangle soup" format that is inefficient for rendering and post-processing [S1]. The method leverages the regular structure of BCC lattices to reduce the complexity of tetrahedral decomposition [S1]. It builds on the principles of tetrahedralization schemes, specifically using a reference tetrahedron and permutation transformations to handle cell types [S1]. The extracted mesh can be used for curvature rendering, leveraging Griffin's technique to estimate Gaussian curvature on the extracted mesh [S1].

## Sources
[S1] Real-Time GPU Technique for Extracting Mesh Isosurfaces from BCC Volume Datasets, https://doi.org/10.15701/kcgs.2020.26.4.17, fetched 2026-10-26
