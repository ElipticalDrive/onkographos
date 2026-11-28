# Sort-middle Parallel Rendering


## Overview
Sort-middle parallel rendering is a distributed rendering strategy that partitions the workload between the geometry processing (transformation and lighting) stage and the rasterization stage [S1, unofficial][S2, unofficial]. Unlike sort-first methods, which distribute primitives before transformation, sort-middle allows geometry processors to handle arbitrary subsets of primitives and then routes the transformed primitives to specific rasterizers based on screen-space coverage [S2, unofficial]. This approach is chosen when high-bandwidth interconnects are available to handle the transfer of transformed primitives, offering a frame-rate advantage over sort-first by avoiding network delays associated with pre-transformation distribution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D primitives (e.g., triangles, voxels) and a set of geometry processors ($G_1 \dots G_m$) and rasterizers ($R_1 \dots R_n$).
*   **Output:** A composite image.

**Data Structures:**
*   **Primitive List:** Unsorted collection of primitives.
*   **Transformed Primitive Buffer:** Stores primitives after transformation and lighting.
*   **Screen Partition Map:** Defines which rasterizer is responsible for which region of the screen.

**Steps:**
1.  **Assignment:** Distribute the input primitives arbitrarily among the available geometry processors [S2, unofficial].
2.  **Geometry Processing:** Each geometry processor $G_i$ performs transformation and lighting on its assigned primitives [S2, unofficial].
3.  **Routing:** Determine the screen-space bounding box or coverage of each transformed primitive. Send each transformed primitive to the rasterizer(s) responsible for the screen regions it overlaps [S2, unofficial].
4.  **Rasterization:** Each rasterizer $R_j$ rasterizes the subset of primitives routed to it [S2, unofficial].
5.  **Compositing:** Combine the results from all rasterizers to form the final image. If rasterizers write to separate framebuffers, a final compositing step merges them [S1, unofficial].

**Pitfalls:**
*   **Load Imbalance:** Primitives may cluster in specific screen regions, causing uneven workloads among rasterizers [S2, unofficial].
*   **Bandwidth:** High bandwidth is required for transferring transformed primitives from geometry processors to rasterizers [S2, unofficial].

## Subtopics

### [Parallel Rendering (Hybrid Systems)](9364-parallel-rendering-hybrid-systems.md)
Combines shared-memory and distributed-memory parallelism to balance scalability and communication overhead.

## Operating characteristics
*   **Performance:** Achieves approximately twice the frame-rate of sort-first approaches by reducing network delays associated with sending primitives before transformation [S1, unofficial].
*   **Scalability:** Scales well with the number of geometry processors, but is limited by the bandwidth of the interconnect between geometry and rasterization stages [S2, unofficial].
*   **Limitations:** Suffers from load imbalance if primitives are unevenly distributed over the screen [S2, unofficial]. Requires high-bandwidth networks for data transfer between stages [S2, unofficial].

## Background
The sort-middle approach was developed to address the inefficiencies of sort-first rendering in distributed systems, such as the Chromium system, where network delays for pre-transformation primitive distribution were significant [S1, unofficial]. Williams and Hiromoto modified Chromium to implement sort-middle, allowing it to achieve higher frame-rates [S1, unofficial]. The approach became popular due to the availability of high-end graphics machines capable of handling the required data transfer rates [S2, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
