# Parallel Projected Tetrahedra with KD-Tree Partitioning

Also called: Parallel Unstructured Volume Rendering in ParaView, Parallel Volume Rendering in ParaView, kd-tree based parallel volume rendering

## Overview
This method enables parallel volume rendering of large unstructured tetrahedral grids by partitioning the data using a kd-tree, allowing the application of a traditional sort-last compositing algorithm [S1]. It renders unstructured grids natively without sampling into a regular grid, avoiding the oversampling/undersampling issues and large sample counts required by grid-based approaches [S1]. Unlike other parallel rendering approaches that require continuous cell transfer, this method requires no further cell transfer once distributed; only image data is communicated between processes [S1].

## How it works
The method extends the [Projected Tetrahedra](877-projected-tetrahedra.md) technique to a distributed-memory parallel environment. It relies on the shared principles of cell projection, specifically the calculation of segment length $l$ and integrated opacity $\alpha$ [S1, S4, book].

### Inputs and Outputs
*   **Input:** An unstructured tetrahedral grid (defined by vertices and connectivity).
*   **Output:** A final composited 2D image.
*   **Parameters:**
    *   Image resolution: 800x800 pixels (used in source experiments) [S1].
    *   Number of partitions: Determined by the number of processes $P$. The kd-tree is built to create $P$ regions [S1].

### Data Structures
1.  **KD-Tree:** A binary tree structure used to partition the spatial domain of the tetrahedral grid. Each leaf node represents a region assigned to a specific process [S1].
2.  **Partitioned Tetrahedra:** The original tetrahedra are split if they intersect with kd-tree region boundaries. This results in a set of clipped tetrahedra assigned to each process. The data size grows during this process [S1].

### Steps
1.  **KD-Tree Construction:** Build a kd-tree over the bounding box of the unstructured grid data. The tree is partitioned to create $N$ regions, where $N$ is the number of available processes [S1].
2.  **Data Partitioning and Splitting:**
    *   Distribute the tetrahedra among the $N$ processes based on the kd-tree regions [S1].
    *   Split any tetrahedron that intersects with a kd-tree region boundary. This ensures that each process holds a complete set of tetrahedra for its region, but causes the total number of tetrahedra to increase [S1].
    *   This partitioning is performed whenever the data changes, not every render frame [S1].
3.  **Distribution:** Transfer the partitioned and clipped tetrahedra to the respective processes. Once distributed, no further cell transfer is required for rendering [S1].
4.  **Rendering:** Each process renders its assigned tetrahedra using the [Projected Tetrahedra](877-projected-tetrahedra.md) method. This involves projecting vertices to screen space, rasterizing the resulting polygons, and computing opacity using the shared principles (specifically $\alpha = 1 - \exp(-\tau_{avg} l)$) [S1, S4, book].
5.  **Parallel Image Compositing (Sort-Last):**
    *   Each process outputs a 2D image (framebuffer) [S1].
    *   Transfer only the image data between processes [S1].
    *   Perform a global sort-last compositing of the images to produce the final result. The specific compositing algorithm is not detailed in the source, but it follows the standard sort-last paradigm where images are sorted and blended [S1].

### Edge Cases and Pitfalls
*   **Memory Usage:** The redistribution process requires holding at least two copies of the data simultaneously (original and partitioned). If hardware memory is insufficient, rendering rates can drop impractically low [S1].
*   **Data Growth:** Splitting cells increases the dataset size. On average, data size grows by about 15% when adding a single level to the kd-tree, with variance between 5% and 30% [S1].
*   **Geometry Changes:** Changes in viewable geometry require lengthy redistribution of the data, which can be a bottleneck [S1].

## Operating characteristics
*   **Time Complexity:** Rendering speed increases with the number of processors, but diminishing returns occur when adding more processors than necessary for a given data set due to overheads like image readback and compositing [S1].
*   **Memory Complexity:** Requires sufficient memory to hold at least two copies of the data during redistribution [S1].
*   **Scalability:** The system shows bigger speedup for larger data sets, even if the overall frame rate is lower [S1].
*   **Strengths:**
    *   Renders unstructured grids natively without sampling [S1].
    *   No further cell transfer after initial distribution; only image data is communicated [S1].
    *   Code reuse: Allows application of existing volume rendering and compositing codes [S1].
*   **Limitations:**
    *   Data redistribution is expensive and requires significant memory [S1].
    *   Changes in viewable geometry require lengthy redistribution [S1].
    *   Rendering rates may be too low for user interaction without level-of-detail techniques [S1].

## Background
This method was developed for ParaView and deployed at Sandia National Laboratories for visualizing simulations from supercomputers like Red Storm [S1]. It addresses the challenge of rendering large quantities of data from high-fidelity physical simulations on distributed-memory parallel computers [S1]. The approach was chosen to avoid the preprocessing time and data size inflation associated with creating hierarchies of grids [S1].

## Subtopics

### Projected Tetrahedra
The base rendering method used by each process. It projects tetrahedra to screen space and rasterizes them. [Projected Tetrahedra](877-projected-tetrahedra.md)

### Parallelized Accurate Sorting for Projected Tetrahedra
An alternative parallel approach that may use different sorting strategies. [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)

### Sorting-free Pre-integrated Projected Tetrahedra
A method that avoids sorting by using pre-integration. [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)

## Sources
[S1] Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
