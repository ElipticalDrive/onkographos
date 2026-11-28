# Hypertree Grid Construction

Also called: HyperTree Grid Resampling/Conversion, Hypertree Grid Generation

## Overview
Hypertree Grid Construction is a preprocessing method that converts unstructured or structured mesh data into a native hypertree grid format to enable efficient visualization of large-scale Adaptive Mesh Refinement (AMR) simulations [S1, unofficial]. It sets itself apart from sibling methods by explicitly building a hierarchical tree structure where coarse cells point to finer sub-grids, allowing full-resolution visualization of datasets with hundreds of millions of cells on consumer hardware, whereas alternatives often require supercomputer nodes for equivalent workloads [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Unstructured mesh data (e.g., VTK, Exodus) or structured Cartesian grid data.
- **Output:** A native hypertree grid data structure optimized for ray-grid traversal.

**Data Structures:**
The hypertree grid consists of a hierarchy of cells. Each cell at level $L$ may have children at level $L+1$.
- **Root Level:** Contains coarse cells covering the domain.
- **Leaf Level:** Contains the finest resolution cells.
- **Internal Nodes:** Represent intermediate refinement levels.
- **Memory Layout:** The structure is stored in a way that minimizes overhead. For unstructured data, the conversion is offline. For structured data, coarser cells are added on top of the final leaves representing the Cartesian grid during streaming/reading [S1, unofficial].

**Steps:**
1. **Data Ingestion:** Load the source mesh data.
2. **Hierarchy Construction:**
   - If the input is **unstructured**: Perform an offline conversion process to build the hypertree structure from the mesh connectivity and geometry [S1, unofficial].
   - If the input is **structured**: During the reading/streaming phase, generate coarser cells that encompass the finest Cartesian grid leaves, effectively wrapping the structured data in a hypertree hierarchy [S1, unofficial].
3. **Optimization:** Ensure the tree structure supports efficient ray-grid intersection by maintaining appropriate bounding boxes for each cell level [G].

**Parameters:**
- **Tree Depth ($\delta$):** The maximum depth of the hypertree. Increasing $\delta$ dilutes the fixed memory costs associated with root-level cells [S1, unofficial].
- **Conversion Mode:** Offline (for unstructured) or Streaming (for structured) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Overhead:** Applying rebalancing filters can cause memory overhead as the input mesh must be retained [S1, unofficial].
- **Load Imbalance:** The cost per cell varies depending on execution parameters and algorithm choice, potentially causing load imbalance in parallel execution [S1, unofficial].

## Subtopics

### Parallel Hypertree Grid Processing
A parallelization strategy for hypertree grid visualization using autonomous processes interacting via message passing, distributing the mesh over processes to improve restitution time and handle large-scale HPC data [S1, unofficial]. It differs from sequential construction by requiring a ghost cell mechanism to copy neighbor information for filters like isocontour or gradient computation [S1, unofficial]. Efficiency depends heavily on the quality of mesh distribution and load balancing, as computational cost per cell is variable [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Memory Footprint:** The hypertree grid object shows almost a full order of magnitude improvement (factor of ~7) in memory footprint compared to unstructured grid representation [S1, unofficial].
- **Execution Time:** Linear speedup in execution time is observed when increasing tree depth $\delta \in [1; 6]$ in a synthetic hypertree grid, demonstrating scalability [S1, unofficial].

**Measured Performance:**
- **Hardware:** Tested on a 16 GB laptop and Tera1000-1 supercomputer nodes (128 GB and 1 TB) [S1, unofficial].
- **Dataset:** The "asteroid fall" dataset (3x10^8 cells) was visualized at full resolution on a 16 GB laptop, replacing the need for multiple supercomputer nodes [S1, unofficial].
- **Scalability:** Strong scalability demonstrated for a 2D AMR mesh with O(10^8) leaves. It scales almost optimally for memory footprint and super-optimally for execution time until maximum speedup is achieved [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Enables interactive exploration of massive datasets on consumer hardware; significant memory reduction compared to unstructured grids [S1, unofficial].
- **Limitations:** Parallel efficiency is sensitive to load balancing; rebalancing filters incur memory overhead [S1, unofficial].

## Background
The method was developed to address the visualization challenges of large-scale HPC simulations, such as the "asteroid fall" simulation provided by Los Alamos National Laboratory/NASA [S1, unofficial]. It was notably applied to allow the exploitation of a 3x10^8-cell dataset on a 16 GB laptop for the first time, moving away from the requirement of several supercomputer nodes [S1, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
