# Pure client-side renderers
Also called: Browser-based volume rendering, Client-side DVR
## Overview
Pure client-side renderers perform all volume rendering computations within the user's web browser, requiring only a file server as a backend to serve volumetric data [S1, unofficial]. This approach eliminates the need for dedicated rendering servers or backend compute infrastructure, shifting the processing load entirely to the client's hardware [S1, unofficial].
## Shared principles
The defining characteristic of this family is the architectural constraint that all rendering logic, including ray casting, compositing, and shading, must execute within the client-side environment (typically via WebGL or WebGPU) [S1, unofficial]. Consequently, the system relies exclusively on a static file server for data delivery, avoiding any server-side computation or gradient calculation [S1, unofficial]. This imposes strict limits on data scale, as rendering is bounded by the client's available GPU memory and computational power [S1, unofficial].
## Subtopics

### [WebSplatter](1741-websplatter.md)
A specific implementation focusing on splatting techniques within the browser.

### [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md)
A unified rendering approach designed for client-side execution.

### [CPU-Driven Task Spawning](4397-cpu-driven-task-spawning.md)
A method for managing rendering tasks using CPU threads within the client environment.

### [Single Node Multi-GPU/Server Architecture](5165-single-node-multi-gpu-server-architecture.md)
An architecture that may involve client-side coordination of multiple GPUs, though typically associated with server-side; included here as a sibling in the broader parallel rendering context.

### [CPU Multithreaded Rendering](6052-cpu-multithreaded-rendering.md)
Utilizing multiple CPU threads for rendering tasks on the client side.

## Background
The concept emerged from the need to visualize large-scale biomedical and scientific data without the overhead of server-side rendering pipelines. Early implementations faced limitations due to available GPU memory, restricting the size of datasets that could be rendered [S1, unofficial]. Notable examples include Viv, a WebGL-based library for biomedical data visualization, and Neuroglancer, which uses a bricked multi-resolution hierarchy to handle large volumetric datasets entirely in the browser [S1, unofficial]. These tools demonstrate the feasibility of client-side rendering for complex data, provided appropriate data structures and optimization techniques are employed.
## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
