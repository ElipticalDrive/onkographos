# Out-of-Core ZSWEEP

Also called: OOC-ZSWEEP

## Overview
Out-of-Core ZSWEEP is a direct volume rendering technique for large unstructured grids that extends the in-core [ZSWEEP](1280-zsweep.md) algorithm to handle datasets exceeding available RAM by processing data in tiles and loading meta-cells from disk only when needed [S1, unofficial]. It differs from its sibling [MIR](1281-mir.md) by being substantially faster and leveraging the ZSWEEP optimization of front-to-back sorting, at the cost of more complex preprocessing and higher main-memory requirements [S1, unofficial].

## How it works
The algorithm is divided into two distinct phases: a view-independent preprocessing phase performed once to generate disk files, and a view-dependent rendering phase [S1, unofficial].

### Inputs and Outputs
*   **Input:** An unstructured volumetric grid defined by vertices and cells.
*   **Output:** A rendered image (pixel colors/compositing results).

### Data Structures
The preprocessing generates two primary files on disk:
1.  **Meta-cell File:** A small file containing a high-level description for each meta-cell. Each entry includes:
    *   Bounding box.
    *   Number of vertices.
    *   Number of cells.
    *   Pointer to the start of the meta-cell's data in the main data file [S1, unofficial].
2.  **Meta-cell Data File:** A larger file containing the actual list of vertices and cells for each meta-cell. Vertices and cells may appear repeated across different meta-cells because each meta-cell is a self-contained unit [S1, unofficial].

During rendering, the following in-memory structures are used:
*   **Pixel Intersection Lists:** For each pixel in the current tile, a sorted list of intersections storing z-values and auxiliary information [S1, unofficial].
*   **Vertex Queue:** A queue used for sweeping vertices. Vertices are tagged as either 'bounding-box' or 'dataset' vertices [S1, unofficial].
*   **Meta-cell Memory Buffer:** Holds the vertices and cells of the currently active meta-cell(s) loaded from disk [S1, unofficial].

### Preprocessing Steps
1.  **Chunk Creation (Meta-cell Generation):**
    *   Let $v$ be the total number of vertices and $m$ be the target number of vertices per meta-cell.
    *   Externally sort all vertices by their x-values.
    *   Partition the sorted vertices into $\lceil v/m \rceil$ consecutive parts.
    *   For each part, externally sort its vertices by y-values.
    *   Partition each part into $\lceil v/m \rceil$ sub-parts.
    *   For each refined part, externally sort its vertices by z-values.
    *   Take the final parts as chunks (meta-cells). These chunks have roughly the same number of vertices but may differ in volume [S1, unofficial].
2.  **Meta-cell Association:**
    *   Compute which cells spatially intersect each meta-cell and identify the vertices belonging to those cells.
    *   This is performed in external memory. The output is the two files described above [S1, unofficial].
    *   *Note:* Unlike the meta-cells in Chiang et al. [2], a cell in this variant can belong to as many meta-cells as it spatially intersects [S1, unofficial].

### Rendering Steps
The rendering is performed tile by tile.

1.  **Tile Division:** Subdivide the screen into tiles [S1, unofficial].
2.  **Tile Processing:** For each tile:
    1.  Compute the set of meta-cells that project into (intersect) the current tile [S1, unofficial].
    2.  Sort these meta-cells in a front-to-back order based on their depth relative to the camera [S1, unofficial].
    3.  Initialize the vertex queue and pixel intersection lists for the tile [S1, unofficial].
    4.  **Sweep Loop:**
        *   While the queue is not empty:
            *   Dequeue the next vertex $v$.
            *   If $v$ is a **bounding-box** vertex:
                *   If the meta-cell $m$ associated with this bounding box has not been loaded:
                    *   Retrieve all vertices and cells of $m$ from the disk files.
                    *   Transform the vertices to view space.
                    *   Insert these vertices into the queue, tagging them as **dataset** vertices [S1, unofficial].
                *   Increment the counter of bounding-box vertices seen for meta-cell $m$.
                *   If the counter for $m$ reaches 8:
                    *   Deallocate the memory for meta-cell $m$ (free memory from disk) [S1, unofficial].
            *   If $v$ is a **dataset** vertex:
                *   Project the faces incident to $v$.
                *   For each pixel covered by the face, compute the intersection of the ray emanating from the pixel with the face.
                *   Store the z-value and auxiliary information in the pixel's intersection list in sorted order [S1, unofficial].
        *   **Compositing:** As the sweep plane reaches the "target Z" plane for each pixel, perform compositing using the accumulated intersection list for that pixel [S1, unofficial].

### Parameters
*   **Target vertices per meta-cell ($m$):** Determines the size of the chunks. The number of meta-cells should be small enough to fit in memory [S1, unofficial].
*   **Tile Size:** Determines the granularity of screen subdivision. Smaller tiles reduce peak memory usage for pixel intersection lists but increase overhead [S1, unofficial].

### Edge Cases and Pitfalls
*   **Memory Usage:** Main memory usage is dominated by pixel intersection lists and the "use set" of each vertex (cells incident to each vertex) for the currently loaded meta-cells [S1, unofficial].
*   **Cell Overlap:** A single cell may be stored in multiple meta-cells if it spatially intersects multiple meta-cells. This increases disk space but simplifies the rendering logic by making meta-cells self-contained [S1, unofficial].
*   **Bounding Box Count:** The algorithm assumes 8 bounding-box vertices per meta-cell (likely corresponding to a hexahedral bounding box). If the meta-cell geometry differs, the deallocation condition must be adjusted [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** Constant memory usage with respect to dataset size. For a large 2048-by-2048 image of the Delta dataset, in-core ZSWEEP requires over 380 MB, while OOC-ZSWEEP requires about 24 MB [S1, unofficial].
*   **Time Complexity:** Rendering times depend on the number of cells and resolution. For the SPX3 dataset at 2048^2 resolution, OOC-ZSWEEP took 1157 seconds to generate the image, with a cost of 174 µs per cell [S1, unofficial].
*   **Comparison with MIR:** OOC-ZSWEEP is usually more efficient than MIR (Memory Insensitive Rendering), sometimes by a factor of 10 or more. However, it requires preprocessing and uses more memory than MIR. MIR is simpler to implement and may be preferred for rendering the same viewpoint with different transfer functions, as lighting calculations are deferred [S1, unofficial].
*   **Strengths:** Enables rendering of datasets larger than RAM; faster than purely disk-based methods like MIR; leverages ZSWEEP's early compositing and sorting optimizations.
*   **Limitations:** Complex preprocessing; higher memory usage than MIR; implementation complexity due to lazy loading and queue management.

## Background
Out-of-Core ZSWEEP was introduced by Farias et al. as an extension of the ZSWEEP algorithm [4] to handle unstructured grids that do not fit in main memory [S1, unofficial]. It builds on the meta-cell concept from Chiang et al. [2], adapting it for the specific needs of the ZSWEEP sweep process. The method was designed to address the limitations of in-core algorithms when dealing with large scientific datasets, providing a balance between the speed of in-core rendering and the capacity of out-of-core techniques [S1, unofficial].

## Sources
[S1] Out-of-core rendering of large, unstructured grids, https://www.researchgate.net/publication/3208958_Out-of-core_rendering_of_large_unstructured_grids, fetched 2026-10-26 (unofficial)
