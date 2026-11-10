# Ray Queue Scheduling

Also called: Cell-Based Ray Queue, Dynamic Ray-Task Scheduling

## Overview
Ray Queue Scheduling is a dynamic load-balancing method for distributed volume rendering that manages ray traversal by associating pending rays with spatial cells (flex-blocks) rather than static image tiles or fixed subvolumes. It solves the problem of uneven computational load caused by variable ray traversal depths and data density, which static distribution cannot address efficiently. Unlike static partitioning, it uses ray queues and neighbor communication to adaptively assign work, ensuring that nodes process data in an order that minimizes cache misses and balances the workload across the cluster [S1, unofficial]. It builds on [Hybrid Image Partitioning](7675-hybrid-image-partitioning.md) by replacing static tile assignment with a demand-driven cell-based queue system.

## How it works
**Inputs and Outputs:**
*   **Input:** A scene with volumetric data distributed using a flex-block partition; a set of primary and spawned rays.
*   **Output:** Rendered image pixels and updated ray states.

**Data Structures:**
*   **Flex-Block (Cell):** A spatial subdivision of the volume. Each cell has defined data extents used for automated space skipping.
*   **Ray Queue:** A data structure holding pending rays for a specific cell. The queue is indexed by cell ID.
*   **Processing State:** A flag indicating whether a cell is currently in main memory and being processed.

**Steps:**
1.  **Initialization:** Distribute the volume data using a flex-block partition. Initialize empty ray queues for each cell.
2.  **Cell Selection:** Choose a cell to process. A greedy max-work approach is used to select the next cell to ensure load balance [S1, unofficial].
3.  **Memory Check:** Determine if the selected cell is in main memory. If not, load it.
4.  **Ray Traversal:** For each ray in the ray queue corresponding to the current cell:
    *   Trace the ray through the cell using space skipping based on the cell's data extents.
    *   If the ray exits the current cell, determine the next cell it intersects.
    *   Place the ray on the ray queue of that next cell.
5.  **Queue Management:** When the queue for the current cell is emptied, return to Step 2 to choose another cell.
6.  **Distribution:** In a distributed setting, ray-task scheduling is updated dynamically within the local partition of each render node. Rays may be communicated to other nodes if they intersect cells managed by other nodes.

**Pitfalls:**
*   Optimizing ray traversal is difficult because the same cell may be revisited by a ray and its descendants any number of times.
*   Descendants may not be generated until the cell has been processed within another cell, leading to potential re-ordering or re-processing requirements [S1, unofficial].

## Subtopics

### Pyramid Clipping Scheduling
A technique by Reinhard et al. using pyramid clipping scheduling and demand-driven scheduling of shadow ray tracing. Each processor handles both types of tasks, with data parallel tasks given higher priority. A pyramid is constructed around a bundle of rays and intersected with an octree subdivision of the volume. Clusters of rays are gathered to improve coherency [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Not given in the sources.

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Adapts to variable ray traversal depths and data density; minimizes overlapping voxel accesses by processing cell-by-cell; improves coherency through ray clustering (in Pyramid Clipping variant).
*   **Limitations:** Complex to optimize due to potential repeated cell visits and dynamic dependency of ray generation on cell processing order.

**When to use it and when not:**
*   Use when load imbalance is significant due to non-uniform data density or complex ray paths (e.g., global illumination).
*   Not given in the sources for when not to use.

## Background
**Origin:**
*   Developed in the context of distributed volume visualization and ray tracing. The approach is described in work related to the Visible Korean data set and massive data management for distributed volume visualization [S1, unofficial].

**History and Development:**
*   The method is part of a hybrid scheme for load balancing in distributed ray tracing, combining flex-block partitioning with dynamic ray-task scheduling. It references earlier work on ray queues [94] and pyramid clipping scheduling by Reinhard et al. [98, 99].

**Where it is used:**
*   Distributed volume visualization systems handling large datasets, such as the Visible Korean data set.

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
