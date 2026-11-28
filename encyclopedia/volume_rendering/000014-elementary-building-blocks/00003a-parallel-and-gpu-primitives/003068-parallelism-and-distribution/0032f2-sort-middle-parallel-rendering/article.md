# Sort-middle Parallel Rendering
## Overview
Sort-middle parallel rendering is a distributed rendering strategy that partitions the workload between the geometry processing (transformation and lighting) stage and the rasterization stage [S2, unofficial]. Unlike sort-first methods, which distribute primitives before transformation, sort-middle assigns arbitrary subsets of primitives to geometry processors, transforms them, and then routes the results to rasterizers based on screen-space coverage [S2, unofficial]. The approach requires high bandwidth for data transfer between the geometry processing and rasterization stages [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D primitives and a set of geometry processors ($G_1 \dots G_m$) and rasterizers ($R_1 \dots R_n$) [S2, unofficial].
*   **Output:** A composite image [G].

**Data Structures:**
*   **Primitive List:** Unsorted collection of primitives [G].
*   **Transformed Primitive Buffer:** Stores primitives after transformation and lighting [G].
*   **Screen Partition Map:** Defines which rasterizer is responsible for which region of the screen [S2, unofficial].
*   **Exact layouts (fields, sizes, bit packing):** Not given in the sources.

**Steps:**
1.  **Assignment:** Distribute the input primitives arbitrarily among the available geometry processors [S2, unofficial].
2.  **Geometry Processing:** Each geometry processor $G_i$ performs transformation and lighting on its assigned primitives [S2, unofficial].
3.  **Routing:** Determine the screen-space bounding box or coverage of each transformed primitive. Send each transformed primitive to the rasterizer(s) responsible for the screen regions it overlaps [S2, unofficial].
4.  **Rasterization:** Each rasterizer $R_j$ rasterizes the subset of primitives routed to it [S2, unofficial].
5.  **Compositing:** Combine the results from all rasterizers to form the final image [S1, unofficial].

**Pitfalls:**
*   **Load Imbalance:** Primitives may cluster in specific screen regions, causing uneven workloads among rasterizers [S2, unofficial].
*   **Bandwidth:** High bandwidth is required for transferring transformed primitives from geometry processors to rasterizers [S2, unofficial].

## Subtopics

### [Parallel Rendering (Hybrid Systems)](9364-parallel-rendering-hybrid-systems.md)
Combines shared-memory and distributed-memory parallelism to balance scalability and communication overhead. It partitions both data and image space to maintain work coherence, reducing the distributed compositing nodes while leveraging high-concurrency shared-memory threads. Choose it when rendering large volumes on clusters of multicore processors or heterogeneous systems to avoid the memory limits of pure shared-memory setups and the communication overhead of pure distributed-memory systems.

## Operating characteristics
*   **Performance:** Achieves approximately twice the frame-rate of sort-first approaches by reducing network delays associated with sending primitives before transformation [S1, unofficial].
*   **Scalability:** Scales with the number of geometry processors, but is constrained by the bandwidth of the interconnect between geometry and rasterization stages [S2, unofficial].
*   **Limitations:** Suffers from load imbalance if primitives are unevenly distributed over the screen [S2, unofficial]. Requires high-bandwidth networks for data transfer between stages [S2, unofficial].

## Background
The sort-middle approach was developed to address the inefficiencies of sort-first rendering in distributed systems, such as the Chromium system, where network delays for pre-transformation primitive distribution were significant [S1, unofficial]. Williams and Hiromoto modified Chromium to implement sort-middle, allowing it to achieve higher frame-rates [S1, unofficial]. The approach became popular due to the availability of high-end graphics machines capable of handling the required data transfer rates [S2, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
