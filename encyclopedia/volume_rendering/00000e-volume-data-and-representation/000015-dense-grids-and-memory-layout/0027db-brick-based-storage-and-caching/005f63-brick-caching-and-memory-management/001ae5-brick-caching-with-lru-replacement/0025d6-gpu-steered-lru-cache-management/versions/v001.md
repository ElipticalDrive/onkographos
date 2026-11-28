# GPU-Steered LRU Cache Management

Also called: Combining Rendering and Visibility, GPU-Steered Data Loading, Unified Management of Brick and Node Pool

## Overview
This method enables the GPU to steer CPU-side data loading for voxel rendering by reporting node usage, thereby eliminating the overhead of CPU-driven visibility queries and tree traversal [S1, unofficial]. It differs from sibling methods by unifying the management of the brick pool and node pool into two LRU-controlled caches where the GPU determines refinement needs and missing data, rather than relying on occlusion queries or frustum containment checks performed on the CPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray traversal results from the GPU (node indices, usage status).
*   **Output:** Updated LRU timestamps in CPU memory; texture updates sent to GPU for new bricks/nodes.

**Data Structures:**
*   **CPU Mirrored Data Structure:** Maintains timestamps for all pool elements (nodes and bricks).
*   **GPU Framebuffer:** Uses Multiple Render Targets (MRTs) to store node usage information.
    *   **Layout:** 3 MRTs are used.
    *   **Capacity:** Each pixel stores $3 \times 4 = 12$ node indices [S1, unofficial].

**Steps:**
1.  **Ray Traversal and Data Collection:** During rendering, each ray traverses the $N^3$-tree. The GPU records the trajectory of used nodes. To handle spatial coherence, rays are grouped into packages (e.g., $2 \times 2$). Within a package, different rays store different sets of the 12 traversed nodes (e.g., upper-left ray stores the first 12, upper-right the second 12) [S1, unofficial]. A $2 \times 2$ package thus collects 48 node indices total ($4 \text{ rays} \times 12 \text{ nodes/ray}$) [S1, unofficial].
2.  **GPU Feedback:** The GPU outputs supplementary information about data usage and update needs alongside the color. This includes which nodes were visited and potentially which need refinement or coarsening [S1, unofficial].
3.  **CPU Update:** The CPU reads the GPU feedback. It updates the timestamps of the corresponding pool elements (nodes and bricks) in its mirrored data structure. Elements with older timestamps are considered for eviction [S1, unofficial].
4.  **Cache Management:** The CPU manages the LRU policy. When a new element is added, it replaces the oldest element (lowest timestamp). The CPU triggers the loading of needed data and stores bricks in the pool using the host-memory timestamps [S1, unofficial].
5.  **Synchronization:** Necessary modifications (new bricks/nodes) are transferred to the GPU via texture-update calls [S1, unofficial].

**Parameters:**
*   **MRT Count:** 3 [S1, unofficial].
*   **Nodes per Pixel:** 12 [S1, unofficial].
*   **Ray Package Size:** $2 \times 2$ (example) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Transparency:** Rays may traverse a large number of nodes, exceeding the fixed output capacity of the framebuffer. The method relies on spatial coherence to mitigate this, but ad-hoc solutions may be needed for extreme cases [S1, unofficial].
*   **Precision:** Timestamps must be reset upon usage to accurately reflect recency [S1, unofficial].

## Subtopics

### Spatial Coherence Ray Grouping
This variant exploits spatial coherence by grouping rays in packages in the image plane to collect node traversal information, reducing the amount of data sent to the CPU [S1, unofficial]. Rays are grouped (e.g., $2 \times 2$), and different node sets are assigned to rays within the package based on their position (e.g., upper-left stores first 12 nodes, upper-right stores second 12 nodes) [S1, unofficial]. A $2 \times 2$ package results in 48 node indices total ($4 \text{ rays} \times 12 \text{ nodes/ray}$) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces CPU overhead by eliminating explicit tree traversal and occlusion queries for visibility [S1, unofficial].
*   **Memory Complexity:** Requires additional GPU memory for MRTs (3 targets per pixel) to store node indices [S1, unofficial].
*   **Strengths:** Unified management of brick and node pools; leverages spatial and temporal coherence; offloads visibility and LOD selection from CPU [S1, unofficial].
*   **Limitations:** Limited by framebuffer output capacity (12 nodes/pixel); may require ad-hoc handling for transparent scenes with long traversal paths [S1, unofficial].
*   **When to use:** Complex scenes where CPU-driven visibility queries are too costly; when spatial coherence is high [S1, unofficial].

## Background
Originated in the paper "Ray-Guided Streaming for Efficient and Detailed Voxel Rendering" (2009) by Crassin et al. [S1, unofficial]. The method was developed to address the significant overhead of CPU-side visibility determination (via occlusion queries and tree traversal) in previous approaches [S1, unofficial]. It builds on the insight that the GPU already possesses traversal information at the end of ray tracing, allowing it to steer data loading efficiently [S1, unofficial].

## Sources
[S1] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
