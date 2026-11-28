# GPU Octree Encoding
Also called: 32-bit Signed Integer Octree Node

## Overview
This method defines the memory layout and traversal optimization for the implicit octree used in VCP (Vector-to-Closest-Point) surface ray-casting. It solves the problem of efficient GPU memory access by encoding the entire octree hierarchy into a compact array of 32-bit signed integers, eliminating explicit pointer indirections. It sets itself apart from standard octree implementations by packing internal node children, empty leaf distances, and non-empty leaf indices into a single signed integer per node, enabling a flat, cache-friendly memory layout [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A constructed VCP octree hierarchy (from [VCP Octree Construction](10082-vcp-octree-construction.md)).
*   **Output:** A flat array of 32-bit signed integers representing the octree nodes, and a separate data structure for non-empty leaf VCP data.

**Data Structures:**
1.  **Octree Array:** A contiguous array of 32-bit signed integers. Each element represents one node.
2.  **VCP Leaf Data:** A separate structure storing the actual VCP vectors for non-empty leaves.

**Node Layout (32-bit Signed Integer):**
The encoding depends on the node type:
1.  **Internal Node:** The integer stores the index of the first child. The 8 children are stored at subsequent locations in the array, from `index` to `index + 7`. The maximum valid index for a child is `0x40000000 - 1` ($2^{30} - 1$) [S1, unofficial].
2.  **Empty Leaf:** The integer is $\ge 0x40000000$ ($2^{30}$). It encodes the minimum distance $d_{min}$ to the nearest VCP vector for all grid points on the boundary of the spatial region covered by this node. The value is computed as:
    $$ j \cdot \min\left\{\frac{d_{min}}{D}, 1\right\} \cdot (2^{30} - 1) + 2^{30} $$
    where $D$ is a cutoff value [S1, unofficial].
3.  **Non-Empty Leaf:** The integer is negative, specifically `-(leafIdx + 1)`. `leafIdx` is an index pointing into the separate VCP leaf data structure [S1, unofficial].

**Construction Parameters:**
*   **Threshold:** Construction stops at a level where the memory requirement falls below a given threshold. This creates an "indirection pool" of nodes rather than a single root node, reducing memory indirections on the GPU [S1, unofficial].

**Optimized Ray Marching (Subtopic Details):**
*   **Node Reuse:** Do not traverse the octree from the root at each step. Remember the node index directly before the leaf. Reuse this index as the starting point, especially when step size is small [S1, unofficial].
*   **Local Memory:** Store the 8 VCP values of each non-empty leaf in local memory to avoid re-accessing global memory while staying in the same cell during small step sizes [S1, unofficial].
*   **Step Size Formula:** Assuming the surface is locally planar, the step size $s$ is:
    $$ s = \frac{d}{\|vcp\| \cdot r} $$
    where $d$ is the distance to the surface along the ray, $\|vcp\|$ is the VCP length, and $r$ is the normalized ray direction [S1, unofficial].
*   **Pitfall:** Using the planar step size formula without restriction leads to rendering artifacts because the surface is not planar on a larger scale. The optimization is restricted by a threshold of $2\epsilon$ to prevent these artifacts [S1, unofficial].

## Subtopics

### Optimized Ray Marching
A traversal optimization strategy for the GPU-encoded octree. It avoids root-to-leaf traversal at every step by caching the last visited node index, reusing it for subsequent steps when the step size is small. It further optimizes by caching the 8 VCP vectors of the current non-empty leaf in local memory. It uses a planar surface assumption to calculate larger step sizes ($s = d / (\|vcp\| \cdot r)$), but restricts this increase by a threshold ($2\epsilon$) to prevent artifacts from non-planar surfaces [S1, unofficial].

## Operating characteristics
*   **Memory Efficiency:** By encoding the tree in a flat array of 32-bit integers and stopping construction at a memory threshold, it minimizes memory indirections and overhead compared to pointer-based trees [S1, unofficial].
*   **Performance:** The optimized ray marching significantly increases frame rates by reducing global memory accesses. Caching leaf VCPs in local memory is crucial when step sizes are very small [S1, unofficial].
*   **Limitations:** The planar surface assumption for step size calculation can cause artifacts if not properly clamped by the $2\epsilon$ threshold [S1, unofficial].

## Background
This encoding method was introduced in the context of Vector-to-Closest-Point (VCP) octrees for surface ray-casting. It addresses the specific needs of GPU rendering where memory bandwidth and indirection costs are critical bottlenecks. The approach builds on the VCP grid construction and adapts the octree storage for efficient parallel traversal [S1, unofficial].

## Sources
[S1] Vector-to-Closest-Point Octree for Surface Ray-Casting, https://doi.org/10.2312/vmv.20151259, fetched 2026-10-26 (unofficial)
