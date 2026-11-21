# Pure client-side renderers
## Overview
Pure client-side renderers perform all volumetric visualization logic within the user's browser, requiring only a static file server for data delivery rather than a dedicated rendering backend [S1, unofficial]. This approach eliminates network latency associated with streaming rendered images or heavy geometry, shifting the computational burden to the client's GPU [S1, unofficial]. Unlike sibling methods that may rely on CPU-driven task spawning or multi-GPU clusters, these renderers are constrained by browser APIs (WebGL/WebGPU) and client memory, necessitating specific optimizations for sorting and data structure mapping [S1, unofficial].
## Shared principles
The articles in this family share the goal of mapping abstract volumetric data to a perceptible output through specific system constraints. Common mechanisms include:
1. **Hardware Acceleration Mapping**: Offloading computationally intensive tasks (ray casting, splatting, sorting) to parallel execution units (GPUs, multi-core CPUs) to achieve real-time or near-real-time performance [S1, S2][S4, unofficial].
2. **Data Structure Optimization**: Storing volumetric data in formats optimized for the target hardware, such as 3D/4D textures for coalesced memory access on GPUs or sparse structures for memory efficiency [S1, unofficial][S3, unofficial].
3. **Depth Management**: Handling the ordering of contributions for alpha-blended rendering, often requiring GPU-based sorting algorithms (radix sort, linked lists) to ensure correct compositing [S1, S2].
4. **Physical-to-Digital Mapping**: In the case of volumetric displays, transforming 3D voxel coordinates into 2D control signals (pixel buffers, SLM patterns) synchronized with mechanical or optical processes to create a physical 3D light field [S1, unofficial][S3, unofficial].
## Subtopics

### [WebSplatter](1741-websplatter.md)
WebSplatter is a WebGPU-based renderer for 3D Gaussian Splatting (3DGS) designed to achieve real-time performance and stability across heterogeneous hardware, ranging from high-end desktop GPUs to low-end integrated graphics and mobile devices. It solves the critical bottlenecks of previous web-based 3DGS implementations, specifically the instability of spin-wait-based sorting on Apple Silicon and the excessive memory consumption that causes crashes on mobile browsers. Unlike its siblings, WebSplatter employs a wait-free GPU-based radix sort and optimized pre-processing pipelines to ensure the system never becomes sort-bound and maintains visual fidelity comparable to native CUDA implementations. See [WebSplatter](1741-websplatter.md).

### [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md)
ClearView (Unified) is a 3-pass WebGL2 renderer designed for the real-time visualization of unstructured hexahedral meshes, specifically targeting wireframe and scalar field rendering in web environments. It solves the problem of rendering complex volumetric data structures without server-side support for Shader Storage Buffer Objects (SSBOs) by encoding mesh geometry into `image2D` textures and using a Parallel Prefix List (PPLL) for fragment management. Unlike sibling methods that may rely on CPU-side sorting or different GPU architectures, ClearView performs all sorting and compositing on the GPU via atomic operations and a final resolve pass, enabling interactive frame rates for large datasets like the SPE-9 benchmark. See [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md).

### Web-Based Volume Rendering
In web-based environments, research focuses on solutions to network latency when loading volumetric data and improving volume rendering performance on the web [S1, unofficial]. Designed to work without a dedicated rendering backend and relies only on a file server. All volume rendering is done in the browser itself [S1, unofficial].

## Background
Pure client-side rendering emerged as a response to the limitations of server-side distributed rendering schemes, which required complex infrastructure for tera-scale volumes [S1, unofficial]. Early web-based volume rendering often offloaded work to remote servers, displaying only the final images in the browser [S1, unofficial]. The shift to pure client-side approaches was driven by the need to eliminate network latency and leverage client-side GPU power, with tools like Neuroglancer demonstrating the viability of WebGL for large-scale volumetric data visualization using brick-based multi-resolution hierarchies [S1, unofficial].
## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
