# GPU-Steered LRU Cache Management
Also called: Combining Rendering and Visibility, GPU-Steered Data Loading, Unified Management of Brick and Node Pool

## Overview
This method enables the GPU to steer data loading for volume rendering by indicating which nodes were helpful, using spatial and temporal coherence to update an LRU cache on the CPU without significant overhead [S1, unofficial]. It solves the problem of CPU-side cache management overhead and visibility query costs by unifying the rendering and visibility update phases, allowing the GPU to dictate brick and node pool states based on actual traversal history [S1, unofficial]. It sets itself apart from sibling methods like [Block-based Ray Cache](2260-block-based-ray-cache.md) by explicitly managing the hierarchical N³-tree node pool alongside the brick pool, and from [LRU Cache Eviction with Residency Update](3115-lru-cache-eviction-with-residency-update.md) by offloading the decision logic to the GPU rather than relying on CPU-driven occlusion queries or simple residency flags [G].

## How it works
**Inputs and Outputs:**
- Input: Rendered image from the current frame, GPU-traversed ray paths, host memory mirrored data structures for node and brick pools [S1, unofficial].
- Output: Updated LRU timestamps for nodes and bricks, texture updates for newly loaded bricks, refined/coarsened tree structure [S1, unofficial].

**Data Structures:**
- **Node/Brick Pool Element:** Contains a `timestamp` field (integer) indicating last usage time [S1, unofficial].
- **Host Mirrored Structure:** A CPU-side copy of the pool state maintained in host memory to track timestamps and manage loading [S1, unofficial].
- **GPU Feedback Buffers:** Three Multi-Render Targets (MRTs) used to store node usage information [S1, unofficial].

**Steps:**
1. **Ray Traversal and Feedback Collection:** During volume rendering, the GPU traverses the N³-tree and records the nodes visited along the entire trajectory for each ray, not just the first hit [S1, unofficial]. This usage data is written to the three MRTs [S1, unofficial].
2. **Spatial Coherence Exploitation:** Neighboring rays visit similar nodes because a brick in a node projects onto several pixels [S1, unofficial]. The GPU aggregates this usage data to determine which nodes are actively used [S1, unofficial].
3. **Timestamp Update on CPU:** The CPU reads the GPU feedback from the MRTs and updates the `timestamp` of the corresponding pool elements (nodes and bricks) in its mirrored data structure [S1, unofficial].
4. **LRU Replacement Decision:** The CPU identifies the oldest elements (lowest timestamps) in the pool [S1, unofficial]. New elements (needed data) replace the oldest ones, and the timestamp of a node/brick is reset upon usage [S1, unofficial].
5. **Data Loading and Texture Update:** The CPU triggers the loading of needed data (bricks) into the pool, determining storage locations via host memory timestamps [S1, unofficial]. All necessary modifications are transferred to the GPU via texture-update calls [S1, unofficial].
6. **Tree Adaptation:** Nodes are subdivided or fused in the N³-tree in an LRU manner based on the usage feedback, effectively managing the node pool as an LRU cache [S1, unofficial].

**Formulas:**
- $C_{pixel} = N_{MRT} \times C_{MRT} = 3 \times 4 = 12$ nodes per pixel, where $N_{MRT}=3$ is the number of MRTs and $C_{MRT}=4$ is the capacity per MRT [S1, unofficial].

**Parameters:**
- **MRTs:** Three Multi-Render Targets are used to store node usage information [S1, unofficial].
- **Nodes per Pixel:** Each pixel can store $3 \times 4 = 12$ nodes [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Missed Nodes:** To ensure no nodes are missed, spatial and temporal coherence are exploited [S1, unofficial].
- **Precision/Boundaries:** Not given in the sources.

## Subtopics

### [Spatial Coherence Ray Grouping](5416-spatial-coherence-ray-grouping.md)
Spatial Coherence Ray Grouping is a technique for optimizing volume rendering by exploiting the spatial coherence of ray traversal paths. It groups neighboring rays into packages (e.g., 2x2 blocks) to collect node traversal information, thereby reducing the volume of data sent to the CPU for cache management. Unlike standard per-ray processing, this method unifies frustum containment, visibility, and Level-of-Detail (LOD) selection within the rendering kernel, offloading workload from the CPU. It is a specific implementation of the broader Ray Aggregation strategy, which traces groups of rays as single units to leverage register SIMD capabilities.

## Operating characteristics
- **Time Complexity:** Reduces CPU-side overhead by eliminating expensive tree traversal and occlusion queries for visibility [S1, unofficial]. GPU-side overhead is introduced for collecting node usage data [S1, unofficial].
- **Memory Complexity:** Requires additional GPU memory for three MRTs to store node usage (12 nodes per pixel) [S1, unofficial]. Host memory mirrors the pool state [S1, unofficial].
- **Performance:** Effective for complex scenes where CPU-driven visibility determination is insufficient [S1, unofficial]. Leverages spatial and temporal coherence to minimize cache misses [S1, unofficial].
- **Strengths:** Unified management of brick and node pools [S1, unofficial].
- **Limitations:** Requires GPU-side logic for feedback collection [S1, unofficial]. Potential for stale data if coherence assumptions fail [S1, unofficial]. Complexity in managing two LRU caches (node and brick) [S1, unofficial].

## Background
Originated in the paper "Ray-Guided Streaming for Efficient and Detailed Voxel Rendering" (CNLE09) by Crassin et al. [S1, unofficial]. The method was developed to address the limitations of CPU-driven visibility determination and occlusion queries, which were found to be too costly for complex scenes [S1, unofficial]. It builds on [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) [G]. It builds on the idea of treating volume data as a rendering primitive and leveraging the GPU's ability to determine visibility during ray traversal [S1, unofficial].

## Sources
[S1] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
