# Sparse Voxel DAG Ray Traversal

Also called: Ray tracing a sparse voxel DAG, SVO DAG Ray Tracer

## Overview
Sparse Voxel DAG (SVDAG) ray traversal is a GPU-based method for efficiently traversing a compressed sparse voxel representation to perform visibility queries (shadows, ambient occlusion) and primary ray intersection [S1]. It builds on [Sparse Voxel DAGs](2791-sparse-voxel-dags.md) by exploiting node sharing to reduce memory footprint while maintaining traversal performance comparable to standard Sparse Voxel Octrees (SVOs) [S1][S2, unofficial]. Unlike standard SVO traversal, it must handle potential cycles in the graph structure by tracking visited nodes, but avoids complex intersection tests with contours [S1].

## How it works
**Inputs:** A Sparse Voxel DAG structure (typically stored in linear memory with child masks and pointers) and a set of rays (primary, shadow, or ambient occlusion rays).
**Outputs:** Intersection distances, hit points, or binary visibility flags.

**Data Structures:**
*   **Node Layout:** Each node consists of a child mask (4 bytes) and a list of pointers to non-empty children. The pointers are stored consecutively after the mask. The size of a node depends on the number of active children [S1].
*   **Visited Set:** A stack or bitset to track nodes currently in the traversal path to detect cycles and prevent infinite loops [G].

**Algorithm Steps:**
1.  **Initialization:** For each ray, start at the root of the DAG. Initialize the traversal stack with the root node.
2.  **Traversal Loop:** While the stack is not empty:
    a.  Pop the current node from the stack.
    b.  **Cycle Check:** If the current node is already in the visited set, skip processing this node (it is an ancestor in the current path) [G].
    c.  **Mark Visited:** Add the current node to the visited set.
    d.  **Child Selection:** Determine which children of the current node are intersected by the ray. This is typically done using a multi-resolution Digital Differential Analyzer (DDA) or axis-aligned bounding box (AABB) tests [S2, unofficial].
    e.  **Push Children:** For each intersected child, push its pointer onto the stack.
    f.  **Termination:** If a leaf node is reached, record the intersection distance or hit point.
3.  **Cleanup:** Remove the current node from the visited set upon backtracking (if using a recursive or explicit stack-based approach that supports backtracking) [G].

**Simplifications for SVDAG vs. SVO:**
*   No intersection tests with contours are needed [S1].
*   Voxels can be traversed in ray order, so distance travelled along the ray does not need to be maintained during the traversal loop itself [S1].
*   The DAG structure requires only minor changes to standard SVO traversal code, primarily the cycle detection mechanism [S1].

**Beam Optimization (Primary Rays):**
*   Use beam optimization with a size corresponding to 8x8 pixels to improve performance [S1].
*   For primary ray intersection evaluation, query a traditional SVO structure containing material information with the hit points obtained from tracing in the DAG structure [S1].

**Soft Shadow Optimization:**
*   Shoot a beam (four rays) per pixel from the view-sample position to the area light [S1].
*   Find the conservative furthest distance along these rays where no geometry is intersected, and shorten all shadow rays accordingly [S1].
*   Trace the same beam rays in reverse direction (from the light source) to find a conservative interval of the beam where geometry may be intersected, further improving traversal speed [S1].
*   This optimization frequently allows finding pixels whose entire beams have no intersection with geometry, requiring no additional shadow rays [S1].

**Ambient Occlusion:**
*   Approximated by casting a number of rays over the hemisphere and averaging the occlusion of each ray [S1].
*   Occlusion is evaluated by mapping the first-hit distance to a falloff function [S1].

## Subtopics

### [GPU Ray-tracing of SSVDAG](1361-gpu-ray-tracing-of-ssvdag.md)
A GPU-based ray traversal method for Sparse Voxel Symmetry-aware Voxel DAGs (SSVDAG) that adapts standard octree traversal to handle variable-rate encoded child pointers and apply symmetry transformations upon entering subtrees. It differs from standard SVDAG traversal by explicitly managing the coordinate system transformations associated with symmetry nodes, enabling efficient visibility queries and primary ray intersection on compressed data. The tracing routine is essentially unchanged from SVO/SVDAG tracing, with small overhead (<15%) to handle the reflective transformations encoded in the tagged pointers. It is used for primary ray tracing and hard shadow ray tracing directly in the SSVDAG structure.

## Operating characteristics
*   **Time Complexity:** Traversal time is proportional to the number of nodes visited. The DAG compression reduces the number of nodes, potentially improving cache efficiency and reducing memory bandwidth usage [S1][S2, unofficial].
*   **Memory Complexity:** Significantly lower than SVO due to node sharing. For example, a scene with nearly 100 billion voxels can be stored in less than 575 MB at 0.048 bits/voxel, compared to 31.1 GB for an SVO [S2, unofficial].
*   **Performance:** Tracing overhead for SSVDAG compared to standard SVO/SVDAG tracing is less than 15% [S2, unofficial].
*   **Strengths:** High compression ratio, efficient visibility queries, suitable for large scenes with repetitive geometry.
*   **Limitations:** Cycle detection adds some complexity. Material information is not stored in the DAG itself and requires a separate structure (e.g., SVO) for shading [S1].

## Background
Sparse Voxel DAGs were introduced to compress sparse voxel octrees by sharing identical subtrees [S1][S2, unofficial]. The ray traversal method described here was implemented in CUDA for efficient GPU rendering [S1]. It builds on the work of Laine and Karras [2010a, 2010b] for SVO traversal and beam optimization [S1]. The SSVDAG variant extends this to handle symmetry transformations [S2, unofficial].

## Sources
[S1] High resolution sparse voxel DAGs, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S2] Symmetry-aware Sparse Voxel DAGs (SSVDAGs) for compression-domain tracing of high-resolution geometric scenes, https://albertojaspe.net/publications/2017-JCGT-ssvdags.pdf, fetched 2026-10-24 (unofficial)
