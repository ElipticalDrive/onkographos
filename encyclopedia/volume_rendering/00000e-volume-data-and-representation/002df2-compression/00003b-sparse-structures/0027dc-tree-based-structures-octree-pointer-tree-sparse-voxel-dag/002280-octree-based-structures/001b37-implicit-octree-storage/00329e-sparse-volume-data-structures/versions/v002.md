# Sparse Volume Data Structures

Also called: Elemental Node Types

## Overview
Sparse Volume Data Structures (SVDS) is a family of hybrid volume data structures that decompose a 3D volume into bricks, assigning each brick one of several "elemental node types" (empty, voxel list, pointer octree, or dense) based on local sparsity to minimize memory consumption [S1, unofficial]. It solves the problem of inefficient memory usage in uniform representations by adapting the storage format to the data density within each brick. Unlike its siblings, which may suffer from branch divergence or pointer-chasing overhead when mixing elemental types, this family (specifically the JIT-compiled variant) can generate specialized code to eliminate dead code and memory fetches, shifting the optimization burden from memory cache to instruction cache [S1, unofficial].

## Shared principles
The core mechanism of this family is the decomposition of the volume into cubic bricks of side length $m$ (where $m$ is a power of two). Each brick is classified into one of four elemental node types based on the number of non-zero elements $n$ and the data size $d$. The goal is to select the type with the minimum memory footprint for each brick, and globally optimize the brick size $m$ [S1, unofficial].

The four elemental node types are:

1.  **Empty:**
    *   **Definition:** Bricks with no non-zero elements.
    *   **Storage:** Omitted from storage; requires zero memory [S1, unofficial].
    *   **Access:** Not applicable.
    *   **Properties:** Allows empty space skipping during traversal [S1, unofficial].

2.  **Voxel List:**
    *   **Definition:** A list of quadruples $(x, y, z, \text{value})$ storing only non-zero elements.
    *   **Storage:** Coordinates are stored as 2-byte integers. Values are stored in a separate buffer. The list is sorted in Morton order [S1, unofficial].
    *   **Memory:** The bit overhead for coordinates is $3 \lceil \log_2(m) \rceil$ bits per element. The upper bound for memory usage is reached if the volume is practically empty (excluding the list structure overhead), and the lower bound if it contains no zero entries [S1, unofficial].
    *   **Access:** $O(n)$ for unsorted lists; $O(\log n)$ for sorted lists using binary search [S1, unofficial].
    *   **Properties:** Memory efficient for extremely sparse volumes. Does not allow empty space skipping within the brick [S1, unofficial].

3.  **Pointer Octree:**
    *   **Definition:** A modified octree where leaf nodes store bricks of size $2^3$ (8 values) instead of single scalars. Internal nodes store 8 pointers.
    *   **Storage:** Pointers are indices into a 1D array. Leaf values are in a separate memory region [S1, unofficial].
    *   **Memory:** A full pointer octree has $(m^3 - 1) / 7$ internal nodes, where $m$ is a power of two. The overhead of pointers is approximately $1/6$ of the number of values in a dense volume [S1, unofficial].
    *   **Access:** $O(\log m)$ traversal steps. Access time depends on the side length $m$, not the number of elements [S1, unofficial].
    *   **Properties:** Shift sensitive; the number of internal nodes in a sparse octree depends on the exact location of the octree within the volume [S1, unofficial]. Allows empty space skipping [S1, unofficial].

4.  **Dense Volume:**
    *   **Definition:** A Cartesian grid storing $m^3$ values per brick.
    *   **Storage:** $d \cdot m^3$ bits, where $d$ is the size of one data entry in bits and $m$ is the brick side length [S1, unofficial].
    *   **Access:** $O(1)$ access time [S1, unofficial].
    *   **Properties:** Lowest overhead per data voxel. Optimal for very dense data. Highest overhead for single values [S1, unofficial].

**Comparison of Elemental Types:**

| Type | Access Time | Bit Overhead / Non-Empty Voxel | Shift Sensitive | Allows Empty Space Skipping |
| :--- | :--- | :--- | :--- | :--- |
| **Empty** | N/A | 0 | N/A | Yes |
| **Voxel List** | $O(\log n)$ | $3 \log_2(m)$ | No | No |
| **Pointer Octree** | $O(\log m)$ | $\sim p/7$ | Yes | Yes |
| **Dense Volume** | $O(1)$ | 0 | No | No |

*Note: $p$ is the pointer size [S1, unofficial].*

## Subtopics

### [Sparse GPU Volume Data Structure with JIT Compilation](6834-sparse-gpu-volume-data-structure-with-jit-compilation.md)
JiTTree is a sparse hybrid volume data structure that combines multiple elemental data structures (empty, voxel list, octree, dense) to minimize memory consumption while using just-in-time (JIT) compilation to generate GPU code tailored to a specific dataset. It solves the problem of increased traversal overhead inherent in conventional hybrid structures by inlining access logic and removing dead code at runtime, shifting the burden from the GPU's memory cache to the instruction cache. Unlike its siblings, which suffer from branch divergence and pointer-chasing overhead when mixing elemental types, JiTTree's traversal overhead depends on the homogeneity of the dataset rather than the number of elemental structures used.

## Background
The concept of using elemental node types for sparse volume data was introduced in the work on "A Just-in-Time Compiled Sparse GPU Volume Data Structure" [S1, unofficial]. The authors identified that single data structures (like pure octrees or dense grids) are suboptimal for volumes with varying sparsity. By combining empty, voxel list, pointer octree, and dense representations, and optimizing them via JIT compilation, they achieved significant memory savings and performance improvements for GPU-based volume rendering [S1, unofficial]. The approach addresses the limitations of traditional hybrid structures by eliminating the overhead associated with generic traversal logic.

## Sources
[S1] A Just-in-Time Compiled Sparse GPU Volume Data Structure, https://vis.uib.no/wp-content/papercite-data/pdfs/Labschuetz-2016-JJC.pdf, fetched 2026-10-29 (unofficial)
