# CUDA-based Exclusive Prefix Sum for Vertex Indexing

Also called: Exclusive scan vertex indexing, Scan-based vertex deduplication

## Overview
This method solves the duplicate vertex calculation problem in GPU-based parallel isosurface extraction by using an exclusive prefix sum (scan) to compute correct global vertex indices for vertices found by parallel threads [S1]. Unlike sibling methods that may only apply stream compaction to triangles, this approach applies scan-based indexing to both vertices and triangles, ensuring that vertices identified by different threads are assigned unique, non-overlapping global indices without requiring expensive post-processing merging [S1]. It builds on [Parallel Prefix Sum (Scan) Orchestration](2593-parallel-prefix-sum-scan-orchestration.md) by providing the specific data structures and kernel orchestration required for volume rendering mesh generation.

## How it works

### Inputs and Outputs
**Inputs:**
- A pair of volume slices (lower and upper) of size $W \times H$ (typically $512 \times 512$ to $2048 \times 2048$) [S1].
- A grid of threads, where each thread processes one cell in the slice pair [S1].

**Outputs:**
- Compacted arrays of unique vertex coordinates (`resvert`) and triangle indices (`restrinumvect`) [S1].
- Global vertex indices assigned to each detected vertex [S1].

### Data Structures
The method requires five sets of matrices for vertex coordinates and labels, and corresponding scan output matrices, plus triangle storage structures [S1]. All matrices are of size $W \times H$, corresponding to the number of cells in the slice pair [S1].

1.  **Vertex Coordinate Matrices:**
    -   `xvert`, `yvert`: Store coordinates for edges along the X and Y axes of the lower slice [S1].
    -   `Xvert`, `Yvert`: Store coordinates for edges along the X and Y axes of the upper slice [S1].
    -   `zvert`: Store coordinates for edges along the Z axis (connecting lower and upper slices) [S1].
2.  **Label Matrices:**
    -   `xlabel`, `ylabel`, `Xlabel`, `Ylabel`, `zlabel`: Corresponding label matrices for the coordinate matrices above [S1]. Initially, all labels are set to 1 to indicate vertex presence [S1].
3.  **Scan Output Matrices:**
    -   `xscan`, `yscan`, `Xscan`, `Yscan`, `zscan`: Output of the exclusive prefix sum operation on the label matrices [S1]. These serve as input for stream compaction and index calculation [S1].
4.  **Triangle Storage:**
    -   `trinumvect`: Stores the number of triangles created for each cell [S1].
    -   `restrinumvect`: Compacted array storing the number of created triangles [S1].
    -   Triangle array size is proportional to slice size, as at most 5 triangles are created per cell [S1].

### Numbered Steps

1.  **Initialization:** For each cell in the slice pair, a dedicated thread executes the `VerticesCalc` kernel. The thread identifies vertices on the cell's edges and writes their coordinates to the respective `xvert`, `yvert`, `Xvert`, `Yvert`, and `zvert` matrices. It also writes a label of 1 to the corresponding label matrix (`xlabel`, etc.) to mark the vertex's presence [S1].

2.  **Exclusive Prefix Sum (Scan):** Perform an exclusive prefix sum operation on each of the five label matrices (`xlabel` through `zlabel`) using the CUDPP library [S1]. The output is stored in the corresponding scan matrices (`xscan` through `zscan`) [S1].
    -   The exclusive scan is defined as:
        $$ V_{out}[i] = \sum_{j=0}^{i-1} V_{in}[j] $$
    -   Example: If `xlabel` contains $\{5, 6, 2, 4, 0, 2, 0, 1\}$, then `xscan` contains $\{0, 5, 11, 13, 17, 17, 19, 19\}$ [S1].

3.  **Index Assignment:** The value in the scan matrix at a given cell's position indicates the number of vertices of that type found in all preceding cells. This value serves as the base global index for the vertices found in the current cell [S1].
    -   If thread $k$ finds $N$ vertices, and the scan output for its cell is $S$, the global indices for these vertices are $S, S+1, \dots, S+N-1$ [S1].
    -   The total number of vertices is obtained by summing the last elements of the input label vector and the output scan vector (e.g., $1 + 19 = 20$) [S1].

4.  **Stream Compaction (Vertices):** Execute the `VerticesCompact` kernel to compact the vertex coordinate matrices into a single contiguous array `resvert` [S1].
    -   The scan output matrices (`xscan` through `zscan`) are used to determine the write positions in `resvert` [S1].
    -   Vertices are stored in a specific order: `xvert`, `yvert`, `Xvert`, `Yvert`, `zvert` [S1].
    -   The global index of a vertex in the final mesh is calculated by summing the counts of previous edge types. For example, if 100 vertices were found previously, and the current cell has 10 X-edges, 20 Y-edges, and 7 Z-edges, the first Z-vertex gets global index $100 + 10 + 20 = 130$ [S1].
    -   However, its position in the compacted `resvert` vector is based on local offsets within the current cell's contribution, specifically position $30$ ($10+20$) relative to the start of the current cell's block in the compacted array [S1].

5.  **Stream Compaction (Triangles):** Execute a second compaction kernel to process the `trinumvect` array into `restrinumvect` [S1]. This step also uses the scan operation to determine write positions for triangle data, ensuring compact storage of triangle indices [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel 1: VerticesCalc
# Each thread processes one cell (i, j) in the W x H slice pair
def VerticesCalc(cell_i, cell_j, volume_lower, volume_upper, 
                 xvert, yvert, Xvert, Yvert, zvert,
                 xlabel, ylabel, Xlabel, Ylabel, zlabel):
    
    # Identify vertices on the 12 edges of the cell
    # Determine if a vertex exists on each edge based on isosurface criteria
    
    # Write coordinates to global memory matrices
    if vertex_exists_on_lower_x_edge:
        xvert[cell_i, cell_j] = compute_vertex_coord(...)
        xlabel[cell_i, cell_j] = 1
    else:
        xlabel[cell_i, cell_j] = 0
        
    if vertex_exists_on_lower_y_edge:
        yvert[cell_i, cell_j] = compute_vertex_coord(...)
        ylabel[cell_i, cell_j] = 1
    else:
        ylabel[cell_i, cell_j] = 0
        
    if vertex_exists_on_upper_x_edge:
        Xvert[cell_i, cell_j] = compute_vertex_coord(...)
        Xlabel[cell_i, cell_j] = 1
    else:
        Xlabel[cell_i, cell_j] = 0
        
    if vertex_exists_on_upper_y_edge:
        Yvert[cell_i, cell_j] = compute_vertex_coord(...)
        Ylabel[cell_i, cell_j] = 1
    else:
        Ylabel[cell_i, cell_j] = 0
        
    if vertex_exists_on_z_edge:
        zvert[cell_i, cell_j] = compute_vertex_coord(...)
        zlabel[cell_i, cell_j] = 1
    else:
        zlabel[cell_i, cell_j] = 0

# Kernel 2: VerticesCompact
# Compacts vertex coordinates and updates indices
def VerticesCompact(cell_i, cell_j, 
                    xvert, yvert, Xvert, Yvert, zvert,
                    xscan, yscan, Xscan, Yscan, zscan,
                    resvert):
    
    # Calculate base indices for this cell based on scan results
    base_x = xscan[cell_i, cell_j]
    base_y = yscan[cell_i, cell_j]
    base_X = Xscan[cell_i, cell_j]
    base_Y = Yscan[cell_i, cell_j]
    base_z = zscan[cell_i, cell_j]
    
    # Determine how many vertices of each type were found in this cell
    count_x = xlabel[cell_i, cell_j]
    count_y = ylabel[cell_i, cell_j]
    count_X = Xlabel[cell_i, cell_j]
    count_Y = Ylabel[cell_i, cell_j]
    count_z = zlabel[cell_i, cell_j]
    
    # Write to compacted array in order: x, y, X, Y, z
    # The position in resvert is determined by the cumulative sum of counts
    # within the current cell's contribution, but the global index is derived from scan
    
    offset = 0
    
    # Write X vertices
    for k in range(count_x):
        resvert[base_x + offset] = xvert[cell_i, cell_j] # Assuming single vertex per edge for simplicity
        offset += 1
        
    # Write Y vertices
    for k in range(count_y):
        resvert[base_y + offset] = yvert[cell_i, cell_j]
        offset += 1
        
    # Write X (upper) vertices
    for k in range(count_X):
        resvert[base_X + offset] = Xvert[cell_i, cell_j]
        offset += 1
        
    # Write Y (upper) vertices
    for k in range(count_Y):
        resvert[base_Y + offset] = Yvert[cell_i, cell_j]
        offset += 1
        
    # Write Z vertices
    for k in range(count_z):
        resvert[base_z + offset] = zvert[cell_i, cell_j]
        offset += 1

# Main Execution Flow
def main():
    # 1. Launch VerticesCalc kernel for all cells
    # Grid size: (W, H, 1)
    VerticesCalc<<<grid, block>>>(...)
    
    # 2. Perform Exclusive Prefix Sum on label matrices using CUDPP
    # CUDPP_scan(xlabel, xlabel, W*H, ADD)
    # CUDPP_scan(ylabel, ylabel, W*H, ADD)
    # ... repeat for Xlabel, Ylabel, zlabel
    
    # 3. Launch VerticesCompact kernel
    VerticesCompact<<<grid, block>>>(...)
    
    # 4. Perform Stream Compaction for triangles (similar scan + compact steps)
    # ...
```

### Parameters
-   **Slice Size:** Common dataset slice sizes are $512 \times 512$ to $2048 \times 2048$ [S1].
-   **Memory Occupancy:** The auxiliary structures consume 78 to 1168 MB of GPU memory depending on slice size [S1].
-   **Minimum GPU Memory:** 512 MB is required to process $512 \times 512$ slices [S1].

### Edge Cases and Pitfalls
-   **GPU Memory Limitation:** The large size of auxiliary data structures for large slices (up to 1168 MB) necessitates processing one pair of slices at a time [S1].
-   **Data Transfer Overhead:** The effective number of vertices and triangles is much lower than the size of the data structures, leading to expensive data transfers. Stream compaction is critical to mitigate this [S1].
-   **Search Structures:** Search structures are disregarded because they also consume GPU memory, which is limited [S1].

## Operating characteristics

-   **Time Complexity:** The method involves four kernels: two for vertex/triangle computation and two for stream compaction and scan operations [S1]. The scan operation is typically $O(N)$ where $N$ is the number of cells [S1].
-   **Memory Complexity:** Memory usage is proportional to the slice size $W \times H$. For $512 \times 512$ to $2048 \times 2048$ slices, memory occupancy ranges from 78 to 1168 MB [S1].
-   **Strengths:**
    -   Solves the duplicate vertex calculation problem efficiently without post-processing merging [S1].
    -   Reduces data transfer overhead via stream compaction [S1].
    -   Compatible with standard CUDA devices with at least 512 MB of memory [S1].
-   **Limitations:**
    -   Memory-intensive, limiting the maximum slice size that can be processed at once [S1].
    -   Data transfer between host and device remains a significant part of the execution time [S1].
-   **When to Use:** Suitable for GPU-based isosurface extraction where memory allows for processing slice pairs and high parallelism is desired [S1].
-   **When Not to Use:** Not suitable for volumes with very large slices that exceed available GPU memory, or when search structures are preferred despite memory costs [S1].

## Background

-   **Origin:** This method was introduced in the paper "A parallel isosurface extraction component for visualization pipelines executing on GPU clusters" [S1].
-   **Development:** It extends previous work [25] which only used stream compaction for triangles, by applying it to both vertices and triangles to address duplicate vertex calculation [S1].
-   **Usage:** It is used in GPU-based visualization pipelines for isosurface extraction from volume data [S1].

## Sources

[S1] A parallel isosurface extraction component for visualization pipelines executing on GPU clusters, https://doi.org/10.1016/j.cam.2014.05.019, fetched 2026-10-27
