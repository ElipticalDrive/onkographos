# Pure client-side renderers
## Overview
Pure client-side renderers perform all volumetric visualization logic within the user's browser, requiring only a file server as a backend rather than a dedicated rendering server [S1, unofficial]. This approach eliminates network latency associated with streaming rendered images and allows for interactive exploration of large datasets directly on the client [S1, unofficial]. Unlike sibling methods that may rely on CPU task spawning or multi-GPU clusters, pure client-side renderers are constrained by browser APIs (WebGL/WebGPU) and client hardware limits, necessitating specific optimizations for memory and sorting [S1, unofficial].
## Shared principles
The articles in this family share the goal of mapping abstract volumetric data to a perceptible output through specific system constraints. Common mechanisms include:
1. **Hardware Acceleration Mapping**: Offloading computationally intensive tasks (ray casting, splatting, sorting) to parallel execution units (GPUs, multi-core CPUs) to achieve real-time or near-real-time performance [S1, S2][S4, unofficial].
2. **Data Structure Optimization**: Storing volumetric data in formats optimized for the target hardware, such as 3D/4D textures for coalesced memory access on GPUs or sparse structures for memory efficiency [S1, unofficial][S3, unofficial].
3. **Depth Management**: Handling the ordering of contributions for alpha-blended rendering, often requiring GPU-based sorting algorithms (radix sort, linked lists) to ensure correct compositing [S1, S2].
4. **Physical-to-Digital Mapping**: In the case of volumetric displays, transforming 3D voxel coordinates into 2D control signals (pixel buffers, SLM patterns) synchronized with mechanical or optical processes to create a physical 3D light field [S1, unofficial][S3, unofficial].
## Subtopics

### [WebSplatter](1741-websplatter.md)
A WebGPU renderer for 3D Gaussian Splatting that uses a wait-free GPU radix sort to avoid spin-waiting on Apple M-series chips, ensuring consistent performance across heterogeneous hardware.

### [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md)
A WebGL2 renderer for unstructured hexahedral meshes that uses a Parallel Prefix List (PPLL) structure in `image2D` textures to manage fragment ordering via atomic operations, avoiding CPU-side sorting.

### [Web-Based Volume Rendering](5039-web-based-volume-rendering.md)
General research into web-based volume rendering, focusing on network latency reduction and performance improvements for volumetric data in browsers.

## Background
The concept of pure client-side rendering emerged as web technologies (WebGL, WebGPU) matured, allowing complex graphics computations to run locally in the browser. Early implementations faced limitations due to GPU memory constraints and lack of efficient sorting mechanisms on the client side [S1, unofficial]. Modern approaches leverage compute shaders and advanced data structures to handle large-scale biomedical and scientific datasets without server-side assistance [S1, unofficial].
## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
