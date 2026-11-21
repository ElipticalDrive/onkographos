# Parallel CPU/GPU Voxelization

Also called: GPU/CPU Parallelization for Voxelization

## Overview
This method accelerates the voxelization of tetrahedral meshes by distributing the computational workload between CPU cores (using OpenMP) and GPU streams (using CUDA) [S1, unofficial]. It addresses the performance bottleneck of iterating over large numbers of tetrahedra and voxel points, which is the core operation in robust voxelization algorithms like Philipp B’s [S1, unofficial]. Unlike pure CPU or pure GPU approaches, this hybrid strategy aims to maximize throughput for high-density meshes while acknowledging the memory transfer overhead that penalizes smaller datasets [S1, unofficial].

## How it works
The method implements a parallel iteration over the tetrahedral mesh elements and the target voxel grid. The specific parallelization logic relies on splitting the workload between CPU threads and GPU kernels [S1, unofficial].

**Inputs:**
*   Tetrahedral mesh data (vertices, tetrahedra connectivity).
*   Target voxel grid resolution (e.g., 256³ binary 3D grid) [S1, unofficial].
*   Parallelization parameters (thread counts for OpenMP, block/grid dimensions for CUDA) [G].

**Outputs:**
*   A voxelized representation of the mesh (binary or signed distance field) [S1, unofficial].

**Data Structures:**
*   **CPU Side:** Arrays storing tetrahedron indices and vertex coordinates.
*   **GPU Side:** Device memory copies of the tetrahedron data and the voxel grid buffer [S1, unofficial].

**Steps:**
1.  **Data Preparation:** Load the tetrahedral mesh data into host memory.
2.  **Decision Logic:** Determine whether to use CPU-only, GPU-only, or hybrid execution based on the tetrahedral count. Small counts may incur overhead that negates GPU speedups [S1, unofficial].
3.  **Data Transfer:** If GPU acceleration is selected, copy the tetrahedron data and voxel grid state to GPU global memory [S1, unofficial].
4.  **Parallel Execution:**
    *   **CPU (OpenMP):** Distribute tetrahedra or voxel points across multiple CPU threads. Each thread checks intersection or containment (e.g., using Philipp B’s algorithm) for its assigned subset [S1, unofficial].
    *   **GPU (CUDA):** Launch a kernel where each thread or block processes a subset of tetrahedra or voxel points. The kernel performs the geometric tests in parallel [S1, unofficial].
5.  **Result Aggregation:** If hybrid, merge results from CPU and GPU. If GPU-only, copy the modified voxel grid back to host memory [S1, unofficial].

**Formulas:**
*   The geometric test for point-in-tetrahedron containment is performed per thread. While the specific formula for Philipp B’s algorithm is not detailed in the provided notes, it is cited as the core geometric primitive [S1, unofficial].
*   Performance gain is measured as $T_{CPU} / T_{CPU+GPU}$ [S1, unofficial].

**Parameters:**
*   Grid Resolution: 256³ (used in benchmarking) [S1, unofficial].
*   Threshold for GPU usage: Not explicitly defined, but performance drops for tetrahedral counts below $10^6$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** Loading data into GPU memory incurs significant overhead. For models with a small tetrahedral count (e.g., $10^5$), the GPU implementation is slower than the CPU-only implementation due to this transfer cost [S1, unofficial].
*   **Small Meshes:** Models with fewer than $10^6$ tetrahedra may not benefit from GPU acceleration [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra and voxel points processed [G].
*   **Measured Performance:**
    *   Large Model ($10^7$ tetrahedra): CPU time 98.931s; CPU+GPU time 9.77s [S1, unofficial].
    *   Medium Model ($10^6$ tetrahedra): CPU time 2.947s; CPU+GPU time 2.776s [S1, unofficial].
    *   Small Model ($10^5$ tetrahedra): CPU time 0.503s; CPU+GPU time 1.665s (GPU is slower) [S1, unofficial].
*   **Strengths:** Significant speedup for large-scale meshes ($>10^6$ tetrahedra) by leveraging GPU parallelism [S1, unofficial].
*   **Limitations:** High overhead for small meshes makes it inefficient for low-poly models [S1, unofficial].
*   **When to use:** High-density tetrahedral meshes where voxelization time is a bottleneck.
*   **When not to use:** Small meshes or scenarios where memory transfer latency dominates computation time.

## Background
This parallelization strategy was implemented to support "Philipp B’s algorithm," a robust method for checking point-in-tetrahedron containment [S1, unofficial]. The algorithm was originally proposed by Philipp Beisel in response to a question by Denis Lolik regarding efficient containment testing [S1, unofficial]. The parallel implementation using OpenMP and CUDA was developed to prove the robustness and scalability of this voxelization approach, specifically addressing the computational cost of sampling voxel points from tetrahedral meshes [S1, unofficial]. The work is part of a broader effort to improve tetrahedral mesh generation and visualization [S1, unofficial].

## Sources
[S1] Robust Voxelization and Visualization by Improved Tetrahedral Mesh Generation, https://doi.org/10.48550/arxiv.2106.01326, fetched 2026-10-26 (unofficial)
