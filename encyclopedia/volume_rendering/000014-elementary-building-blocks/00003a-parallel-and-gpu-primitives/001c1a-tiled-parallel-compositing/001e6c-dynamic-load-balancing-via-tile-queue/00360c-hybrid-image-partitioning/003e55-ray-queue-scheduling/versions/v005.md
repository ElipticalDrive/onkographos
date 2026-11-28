# Ray Queue Scheduling

Also called: Cell-Based Ray Queue, Dynamic Ray-Task Scheduling

## Overview
Ray Queue Scheduling is a hybrid load-balancing scheme for distributed ray tracing that dynamically updates task assignments within the local partition of each render node [S1, unofficial]. It addresses uneven computational loads by processing one cell at a time and routing rays to the queues of their next intersected cells [S1, unofficial]. The method builds on [Hybrid Image Partitioning](7675-hybrid-image-partitioning.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A distributed volumetric scene partitioned into flex-blocks (cells); a set of active rays; local memory of each render node [S1, unofficial].
*   **Output:** Completed ray traces; updated ray queues reflecting the new positions of rays in the scene [S1, unofficial].

**Data Structures:**
*   **Flex-blocks:** The spatial subdivision units, each with defined data extents [S1, unofficial].
*   **Ray Queue:** A data structure holding pending rays for each cell [S1, unofficial].
*   **Cell State:** Indicates whether a cell is in main memory and ready for processing [S1, unofficial].

**Steps:**
1.  **Initialization:** Distribute the initial set of rays into the ray queues of the cells they initially intersect [G].
2.  **Cell Selection:** Choose a cell to process using a greedy max-work approach [S1, unofficial].
3.  **Ray Traversal:** If the selected cell is in main memory, trace all rays currently in its corresponding ray queue through the cell [S1, unofficial]. Space skipping is automated using the data extents of the flex-block [S1, unofficial].
4.  **Ray Propagation:** When a spawned ray exits the current cell, determine the next cell it intersects and place the ray on that cell's queue [S1, unofficial].
5.  **Queue Management:** If the queue for the current cell is emptied, return to Step 2 to choose another cell [S1, unofficial].
6.  **Iteration:** Repeat until all rays are completed or the rendering pass is finished [G].

**Pitfalls and Edge Cases:**
*   **Revisits:** The same cell may be revisited by a ray and its descendants any number of times [S1, unofficial].
*   **Delayed Generation:** Descendants may not be generated until the cell has been processed within another cell, complicating optimization [S1, unofficial].
*   **Scheduling Difficulty:** Optimizing ray traversal is difficult due to the dynamic and non-deterministic nature of ray generation and cell revisits [S1, unofficial].

## Subtopics

### Pyramid Clipping Scheduling
A technique by Reinhard et al. that combines pyramid clipping with demand-driven scheduling of shadow ray tracing [S1, unofficial]. Each processor handles both data parallel and shadow ray tasks, with data parallel tasks given higher priority [S1, unofficial]. A pyramid is constructed around a bundle of rays and intersected with an octree subdivision of the volume to determine relevant cells [S1, unofficial]. Clusters of rays are gathered to improve coherence [S1, unofficial].

### [Dynamic Task Scheduling](9785-dynamic-task-scheduling.md)
Dynamic Task Scheduling is a load-balancing method that dynamically schedules subvolumes or ray tasks within each render node. It addresses the NP-Complete problem of determining the optimum order in which cells are cached by adaptively assigning work using ray queues and neighbor communication. It is chosen for distributed ray tracing contexts and deferred workload mechanisms in game engines, and for GPU rendering of Kernel-Reflection Sequences where it breaks ray evaluation into variable-length tasks to handle differing convergence times. See [Dynamic Task Scheduling](9785-dynamic-task-scheduling.md).

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Not given in the sources [S1, unofficial].
*   **Memory:** Requires memory for ray queues per cell and the storage of flex-blocks in main memory when processed [S1, unofficial].

**Performance and Scaling:**
*   **Measured performance:** Not given in the sources [S1, unofficial].
*   **Scaling:** Not given in the sources [S1, unofficial].
*   **Strengths:** Handles uneven computational loads in distributed environments [S1, unofficial].
*   **Limitations:** Optimization is difficult due to frequent cell revisits and delayed descendant generation [S1, unofficial].
*   **When to Use:** Not given in the sources [S1, unofficial].
*   **When Not to Use:** Not given in the sources [S1, unofficial].

## Background
Originating from research on distributed volume visualization, the method addresses load balancing challenges in distributed ray tracing systems [S1, unofficial]. It builds on the concept of flex-block partitioning and dynamic task scheduling to handle massive datasets like the Visible Korean data set [S1, unofficial]. The approach is part of a broader framework for volumetric ray tracing that includes global illumination effects [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
