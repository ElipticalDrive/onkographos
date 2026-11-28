# Hybrid kd-tree/Uniform Grid
Also called: Hybrid kd-tree/uniform grid structure (Zellmann et al.), Zellmann et al. Hybrid Structure

## Overview
This method accelerates volume rendering by combining a kd-tree at the root level with a uniform grid at the leaf-node level to perform empty-space skipping [S1, unofficial]. It addresses the trade-off between the high construction cost of deep trees and the coarse skipping granularity of shallow trees by decoupling tree depth from skipping granularity, allowing for faster construction while maintaining efficiency in large datasets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The primary volume dataset (voxel values), a transfer function, and the hybrid acceleration structure.
*   **Output:** A rendered image with inactive regions skipped.

**Data Structures:**
1.  **Hybrid Structure:**
    *   **Root:** A kd-tree. Each node represents a spatial region.
    *   **Leaves:** Each leaf node of the kd-tree contains a uniform grid (macro-cell grid). This grid subdivides the leaf's spatial region into smaller cells.
    *   **Metadata:** Each cell in the uniform grid stores occupancy information (e.g., whether the region contains active data above a threshold $\tau$).

**Steps:**
1.  **Construction:**
    *   Build a kd-tree over the volume space.
    *   For each leaf node of the kd-tree, construct a uniform grid covering the leaf's bounding box.
    *   Populate the uniform grid cells with occupancy data based on the volume content within that leaf's region [S1, unofficial].
2.  **CPU Traversal:**
    *   Traverse the kd-tree on the CPU.
    *   Identify and collect all non-empty leaf nodes (those containing active data) [S1, unofficial].
    *   Sort the list of non-empty leaf nodes (e.g., by depth or ray intersection order) [S1, unofficial].
3.  **GPU Rendering:**
    *   Iterate over the sorted list of non-empty leaf nodes on the GPU.
    *   For each active leaf, traverse its internal uniform grid to determine fine-grained active regions.
    *   Perform ray marching/sampling only within these active regions [S1, unofficial].

**Parameters:**
*   **Threshold $\tau$:** Not specified in sources. Typically a density or opacity threshold to define "active" data.
*   **Leaf Grid Resolution:** Not specified in sources. Determines the granularity of empty-space skipping within each kd-tree leaf.

**Edge Cases and Pitfalls:**
*   **Transfer Function Changes:** If the acceleration structure is transfer-function dependent (as noted in related approaches like Chebychev distance maps [S1, unofficial]), it must be updated when the transfer function changes. The sources do not explicitly state if this hybrid structure is TF-dependent, but caution is advised [S1, unofficial].
*   **CPU-GPU Synchronization:** The method requires transferring the sorted list of non-empty leaves from CPU to GPU, introducing potential synchronization overhead [S1, unofficial].

## Subtopics

### [Hybrid Grids](11219-hybrid-grids.md)
A strategy combining a shallow k-d tree with a global grid of macro cells. It solves the trade-off between high construction cost of deep trees and coarse skipping granularity of shallow trees by using the k-d tree for coarse traversal and the global grid for fine-grained empty space detection within leaf nodes. It sets itself apart by decoupling tree depth from skipping granularity.

## Operating characteristics
*   **Time Complexity:** Construction cost is lower than deep trees due to the shallow kd-tree and uniform grids [S1, unofficial]. Rendering speed depends on the efficiency of CPU traversal and GPU iteration over sorted leaves.
*   **Memory Complexity:** Requires storage for the kd-tree nodes and the uniform grids within each leaf.
*   **Performance:** Not given in the sources.
*   **Strengths:** Faster construction than deep hierarchical structures; fine-grained skipping within leaves [S1, unofficial].
*   **Limitations:** Requires CPU-GPU data transfer; potential need for updates if transfer-function dependent [S1, unofficial].
*   **When to use:** Large datasets where construction time is a bottleneck and fine-grained skipping is needed within regions [S1, unofficial].
*   **When not to use:** When transfer functions change frequently and the structure is TF-dependent [S1, unofficial].

## Background
Originated from Zellmann et al. [ZML19], adapting previous work on summed-volume tables for kd-tree construction [S1, unofficial]. The approach was designed for GPU-based construction and rendering [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
