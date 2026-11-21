# Sparse Voxel DAG Ray Traversal

Also called: Ray tracing a sparse voxel DAG, SVO DAG Ray Tracer

## Overview
Sparse Voxel DAG (SVDAG) ray traversal is a GPU-based method for efficiently traversing a compressed sparse voxel representation to perform visibility queries (shadows, ambient occlusion) and primary ray intersection [S1]. It builds on [Sparse Voxel DAGs](2791-sparse-voxel-dags.md) by exploiting node sharing to reduce memory footprint while maintaining traversal performance comparable to standard Sparse Voxel Octrees (SVOs) [S1][S2, unofficial]. Unlike standard SVO traversal, it handles shared nodes where multiple parents reference the same child, but avoids complex intersection tests with contours [S1].

## How it works
**Inputs:** A Sparse Voxel DAG structure stored in linear memory and a set of rays (primary, shadow, or ambient occlusion) [S1].
**Outputs:** Intersection distances, hit points, or binary visibility flags [G].
**Data Structures:** Each node consists of a 4-byte child mask followed consecutively by pointers to non-empty children; node size scales with the number of active children [S1].
**Algorithm Steps:**
1. Initialize the traversal stack with the root node for each ray [G].
2. While the stack is not empty, pop the current node [G].
3. Determine which children of the current node are intersected by the ray using a multi-resolution Digital Differential Analyzer (DDA) or axis-aligned bounding box tests [S2, unofficial].
4. For each intersected child, push its pointer onto the stack [G].
5. If a leaf node is reached, record the intersection distance or hit point [G].
6. Remove the current node from the traversal context upon backtracking [G].
**Formulas:** Not given in the sources.
**Parameters:** Beam size is 8×8 pixels for primary rays; soft shadow beams use four rays per pixel [S1].
**Edge Cases and Pitfalls:** DAGs are acyclic, so cycle detection is unnecessary; however, shared nodes require careful stack management to avoid redundant processing [G]. Material properties are not stored in the DAG, requiring a secondary SVO query for shading [S1].

**Simplifications for SVDAG vs. SVO:** No intersection tests with contours are required, and voxels are traversed in strict ray order, eliminating the need to maintain the distance travelled along the ray during the loop [S1]. The DAG topology requires only minor modifications to standard SVO traversal code [S1].

**Beam Optimization (Primary Rays):** Primary rays use an 8×8 pixel beam optimization to improve performance [S1]. For primary ray intersection evaluation, hit points from the DAG are used to query a traditional SVO structure containing material information [S1].

**Soft Shadow Optimization:** A beam of four rays is shot per pixel from the view-sample position to the area light [S1]. The conservative furthest distance along these rays where no geometry is intersected is found, and all shadow rays are shortened accordingly [S1]. Tracing the same beam rays in reverse from the light source identifies a conservative interval where geometry may be intersected, further improving traversal speed [S1]. This frequently allows pixels with no beam intersections to skip additional shadow rays entirely [S1].

**Ambient Occlusion:** Approximated by casting rays over the hemisphere and averaging occlusion, mapping the first-hit distance to a falloff function [S1].

## Subtopics

### [GPU Ray-tracing of SSVDAG](1361-gpu-ray-tracing-of-ssvdag.md)
A GPU-based ray traversal method for Sparse Voxel Symmetry-aware Voxel DAGs (SSVDAG) that adapts standard octree traversal to handle variable-rate encoded child pointers and apply symmetry transformations upon entering subtrees. It differs from standard SVDAG traversal by explicitly managing the coordinate system transformations associated with symmetry nodes, enabling efficient visibility queries and primary ray intersection on compressed data. Choose this variant when rendering scenes with high geometric repetition or reflective symmetry that benefit from lossless compression without material property storage.

## Operating characteristics
**Time Complexity:** Traversal time is proportional to the number of visited nodes; DAG compression reduces the node count, improving cache efficiency and memory bandwidth usage [S1][S2, unofficial].
**Memory Complexity:** Significantly lower than SVO due to node sharing; a ~100 billion voxel scene fits in <575 MB (0.048 bits/voxel), versus 31.1 GB for an SVO [S2, unofficial].
**Measured Performance:** Tracing overhead for SSVDAG compared to standard SVO/SVDAG is <15% on GPU hardware [S2, unofficial].
**Strengths:** High compression ratio, efficient visibility queries, suitable for large scenes with repetitive geometry [S1][S2, unofficial].
**Limitations:** Material properties are not stored in the DAG and require a separate structure (e.g., SVO) for shading [S1].
**When to use:** Use for primary ray intersection, hard shadows, and ambient occlusion on highly repetitive geometric scenes where memory bandwidth is a bottleneck [S1][S2, unofficial].
**When not to use:** Avoid when scenes require per-voxel material or reflectance properties to be stored directly in the traversal structure [S2, unofficial].

## Background
Sparse Voxel DAGs were introduced to compress sparse voxel octrees by sharing identical subtrees [S1][S2, unofficial]. The ray traversal method was implemented in CUDA for efficient GPU rendering [S1]. It builds on Laine and Karras [2010a, 2010b] for SVO traversal and beam optimization [S1]. The SSVDAG variant extends this to handle symmetry transformations [S2, unofficial].

## Sources
[S1] High resolution sparse voxel DAGs, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S2] Symmetry-aware Sparse Voxel DAGs (SSVDAGs) for compression-domain tracing of high-resolution geometric scenes, https://albertojaspe.net/publications/2017-JCGT-ssvdags.pdf, fetched 2026-10-24 (unofficial)
