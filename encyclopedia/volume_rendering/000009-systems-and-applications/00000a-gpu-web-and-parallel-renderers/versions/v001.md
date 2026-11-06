# GPU, web and parallel renderers


## Overview
This family encompasses rendering systems that offload volume rendering computations to parallel hardware (GPUs, multi-core CPUs) or execute entirely within web browsers using WebGL/WebGPU. It addresses the computational intensity of volume rendering by leveraging massive parallelism for tasks like ray casting, splatting, and data sorting. It is distinguished from [Voxel engines and games](47-voxel-engines-and-games.md) by its focus on general-purpose visualization rather than game engine integration, and from [Film and production rendering](48-film-and-production-rendering.md) by its emphasis on real-time or near-real-time interactivity and web compatibility.

## Shared principles
The core mechanism involves mapping volumetric data processing onto parallel execution units. Common principles include:
1.  **Parallel Data Access**: Volumetric data is often stored in textures (3D/4D) or structured buffers (OpenVDB, sparse grids) optimized for coalesced memory access on GPUs [S1].
2.  **Depth Sorting**: For alpha-blended rendering (e.g., splatting), fragments or splats must be sorted by depth. This is often the bottleneck, addressed via GPU-based radix sorts or linked lists [S1].
3.  **WebGL/WebGPU Constraints**: Web-based renderers must operate within the limitations of browser APIs, such as the lack of Shader Storage Buffer Objects (SSBOs) in WebGL2, requiring textures for read/write operations and atomic counters [S1].
4.  **Multi-GPU/Stream Optimization**: Advanced implementations use multiple CUDA streams or multi-GPU contexts to overlap rendering stages (e.g., left/right eye in VR) or balance load, though synchronization overhead must be managed [S2].

## Subtopics

### [WebSplatter](1741-websplatter.md)
WebSplatter is a WebGPU-based renderer for 3D Gaussian Splatting (3DGS) designed to achieve high-performance, stable, and cross-device compatibility for web-based applications. It solves the critical bottlenecks of previous web-based 3DGS implementations, specifically addressing the instability of Firefox WebGPU, excessive memory consumption on mobile devices, and severe performance degradation due to spin-wait-based sorting on Apple Silicon architectures. WebSplatter distinguishes itself from its siblings by employing a wait-free GPU-based radix sort and a highly optimized pre-processing pipeline that prevents the renderer from becoming sort-bound, ensuring consistent frame times and low variance across a wide spectrum of hardware, from high-end desktops to low-end integrated graphics and mobile phones.

### [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md)
ClearView (Unified) is a 3-pass WebGL2 renderer for unstructured hexahedral meshes that uses a Parallel Prefix List (PPLL) to render wireframes and scalar fields on the GPU. It solves the problem of rendering complex volumetric meshes in web browsers by adapting a desktop GPU technique (which typically relies on Shader Storage Buffer Objects) to WebGL2's texture-only data storage model. It is distinguished from its siblings by its explicit 3-pass architecture (Clear, Gather, Resolve) and its use of atomic operations on image textures to build a linked list of fragments for depth-sorted alpha blending.

### [CPU-Driven Task Spawning](4397-cpu-driven-task-spawning.md)
A model where the CPU decides when and how GPU tasks are launched, treating the GPU as a coprocessor that executes work requests originating from the CPU but cannot generate new work requests on its own.

### [Pure client-side renderers](5040-pure-client-side-renderers.md)
Pure client-side renderers only require a file server as a backend.

### [Single Node Multi-GPU/Server Architecture](5165-single-node-multi-gpu-server-architecture.md)
Presents a similar approach using a single node server with multiple GPUs and CPUs instead.

### Multi-GPU Load Balancing
- **Step**: Create a GL-Context on both X-screens (for multiple NVidia cards) and switch rendering between them one time per frame [S1]. Combine results of both cards. Since the second card's result is combined with the first, the second card need not be connected to a monitor [S1]. Split the sorted array of voxels equally over both cards by interleaving detail coefficients (and histogram) for each pyramid level [S1]. Combine resulting images using a pixel-wise maximum operator [S1].
- **Parameter**: Preferred pixel format for CPU transfer: `GL_UNSIGNED_INT_8_8_8_8` [S1].
- **Pitfall**: No extension exists for direct data shuffling between cards; intermediate results must pass through the CPU using `glReadPixels` on one context and `glDrawPixels` on the other [S1].
- **Comparison**: Multi-threading rendering loops for multiple contexts did not yield performance gains because the algorithm is GPU-bound; CPU synchronization overhead outweighs the tiny load [S1]. Interleaving coefficients is preferred over splitting in the middle to better divide load during progressive refinement [S1].

### Concurrent CUDA Streams for VR
- **Definition**: A VR optimization that concurrently runs CUDA kernels for the left and right eye in multiple streams to better utilize the GPU and mitigate under-utilization caused by imbalanced workloads (varying amounts of splats per tile) [S2]. Overlapping rendering pipelines for left and right eyes in VR using separate, non-blocking CUDA streams to fill GPU utilization gaps caused by imbalanced workloads [S2].
- **Step**: Overlap the right eye's tile ID sorting and rasterization stages with the left eye's rasterization stage [S2].
- **Pitfall**: Imbalanced workloads under-utilize the GPU because the stream waits until the kernel's longest-running thread-blocks are finished [S2]. Do not overlap the left and right eye's sorting stages, as this would require creating separate intermediate sorting buffers, increasing memory usage [S2].
- **Number**: On the Berlin scene (1.0 M splats), single stream takes 6.3 ms while concurrent streams take 5.2 ms. On the Campus scene (28.6 M splats), single stream takes 16.2 ms while concurrent streams take 13.8 ms [S2].

### Web-Based Volume Rendering
- **Definition**: In web-based environments, research focuses on solutions to network latency when loading volumetric data and improving volume rendering performance on the web [S3, unofficial]. Designed to work without a dedicated rendering backend and relies only on a file server. All volume rendering is done in the browser itself [S3, unofficial].

### CPU Multithreaded Rendering
- **Step**: The CPU software version splits the world in sections for rendering in parallel [S4].
- **Parameter**: Performance tested on CPU systems ranging from 1 core to 32 cores (Opteron) [S4].

### Neuroglancer Multi-channel Rendering
- **Definition**: Uses the same technique as Viv for representing multiple channels on the GPU (accumulation-level intermixing) [S3, unofficial].

## Background
Web-based volume rendering research addresses network latency and browser performance, often relying on pure client-side architectures that require only a file server [S3, unofficial]. Early implementations focused on offloading rendering to remote servers, but modern approaches like WebSplatter and ClearView leverage WebGPU and WebGL2 for local computation [S1][S3, unofficial]. Multi-GPU techniques using NVidia drivers allow splitting workloads across cards via X-screens, though CPU-mediated data transfer remains a bottleneck [S1]. VR rendering optimizations utilize concurrent CUDA streams to overlap left/right eye rendering, improving utilization on imbalanced workloads [S2]. CPU-based multithreaded raycasting has been tested on systems up to 32 cores, showing linear scaling potential [S4].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
[S3] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S4] Voxel Game Engine, http://voxelgames.com/, fetched 2026-10-27
