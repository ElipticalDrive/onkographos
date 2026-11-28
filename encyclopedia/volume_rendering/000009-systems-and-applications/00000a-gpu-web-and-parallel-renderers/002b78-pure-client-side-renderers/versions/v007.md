# Pure client-side renderers
## Overview
Pure client-side renderers execute all volumetric rendering computations within the web browser, requiring only a file server as a backend [S1, unofficial]. Unlike server-side rendering schemes that stream rendered images, this approach performs volume rendering directly in the browser itself [S1, unofficial].
## Shared principles
Builds on the shared principles of [GPU, web and parallel renderers](50-gpu-web-and-parallel-renderers.md) [S1, unofficial].
## Subtopics

### [WebSplatter](1741-websplatter.md)
WebSplatter is a pure client-side WebGPU renderer for 3D Gaussian Splatting (3DGS) designed to achieve high-performance, cross-device compatibility in web browsers. It solves the problem of inconsistent performance and stability in existing web-based 3DGS viewers by replacing CPU-based or spin-wait-based sorting with a fully GPU-based, wait-free radix sort. Unlike its sibling [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md), which focuses on unified rendering pipelines, WebSplatter specifically targets the constraints of the WebGPU API and heterogeneous hardware (from high-end desktops to mobile phones), ensuring that the system never becomes sort-bound and maintains visual fidelity comparable to native CUDA implementations.

### [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md)
ClearView (Unified) is a 3-pass WebGL2 renderer designed for unstructured hexahedral mesh visualization, specifically targeting wireframes and scalar fields. It solves the problem of rendering complex volumetric meshes in the browser by replacing traditional CPU-side sorting or SSBO-based GPU buffers with a Parallel Prefix List (PPLL) structure stored in image2D textures. Unlike [WebSplatter](1741-websplatter.md), which uses splatting techniques, ClearView explicitly rasterizes per-face triangles and manages fragment ordering via atomic operations and a linked-list-like buffer, enabling correct alpha blending without depth-sorting artifacts.

### [Web-Based Volume Rendering](5039-web-based-volume-rendering.md)
Web-Based Volume Rendering refers to the class of pure client-side volume rendering systems that execute entirely within a web browser, requiring only a file server as a backend. It solves the problem of network latency and server infrastructure costs associated with remote rendering by offloading all computational work to the client. Unlike server-side rendering schemes that stream rendered images, this approach leverages client-side GPU acceleration to render volumetric data directly in the browser, distinguishing it from siblings like [WebSplatter](1741-websplatter.md) which may employ specific splatting optimizations, and [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md) which offers a unified rendering pipeline.

## Background
Pure client-side rendering emerged to avoid the need to compute gradients on the client and to work without a dedicated rendering backend [S1, unofficial]. Early implementations relied on WebGL to enable hardware-accelerated volume rendering in browsers, allowing for the visualization of biomedical and scientific data without dedicated backend servers [S1, unofficial]. Notable examples include Neuroglancer, which uses a bricked multi-resolution hierarchy to scale to large datasets, and Viv, a WebGL-based library for biomedical data visualization [S1, unofficial].
## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
