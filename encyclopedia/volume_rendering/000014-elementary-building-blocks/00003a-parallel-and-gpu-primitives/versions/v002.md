# Parallel and GPU primitives

Also called: Parallel execution primitives, GPU parallelism, Distributed rendering primitives

## Overview
This family encompasses the fundamental computational mechanisms used to map volume rendering tasks—such as ray marching, compositing, and data structure construction—onto parallel hardware architectures, primarily GPUs and multi-core CPUs. It solves the computational bottleneck of processing millions of rays or voxels by distributing work across thousands of threads or processors. What sets this family apart from its siblings is its focus on the *execution model* and *data movement* primitives (e.g., atomics, shuffles, interop) rather than the geometric algorithms themselves (like ray-primitive intersection) or the high-level rendering strategies (like tiled compositing). It provides the building blocks for implementing the methods described in [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md), [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md), and others.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model.

**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model.

**Parallel Execution**
Computations are mapped to parallel threads. Each thread typically processes one or more pixels or samples. Memory access patterns are optimized for coalescing. Synchronization is used to manage data dependencies between threads.

## Subtopics

### [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md)
A rendering strategy that divides the framebuffer into disjoint spatial tiles to enable lock-free, zero-synchronization parallel processing of semi-transparent primitives (such as splats or Gaussians). It solves the concurrency conflict inherent in alpha compositing by ensuring that each thread writes exclusively to its assigned tile's pixels, eliminating the need for atomics or locks. Unlike [Parallel Direct Compositing](3201-parallel-direct-compositing.md), which may require global synchronization or atomic operations for overlapping writes, tiled compositing relies on spatial partitioning to guarantee data consistency.

### [Compute Shader Kernel](1202-compute-shader-kernel.md)
A GPU compute shader kernel is a parallel execution primitive that maps computational work to threads using thread IDs. In the provided example, it generates a gradient texture by computing normalized coordinates per thread and writing them to a 2D output texture.

### [CUDA-OpenGL Interoperability](1738-cuda-opengl-interoperability.md)
CUDA-OpenGL Interoperability is a mechanism that allows CUDA kernels to read from and write to OpenGL resources (textures, buffers) directly in GPU memory, eliminating CPU-mediated data transfers. It solves the bottleneck of copying rendered volume data or simulation results from CUDA device memory back to system memory for display. It sets itself apart from standard interop methods by supporting both read-only access (via mapped arrays/texture objects) and write access (via surface objects or direct buffer pointers), enabling zero-copy rendering pipelines.

### [Y-Bucket Parallelization](2108-y-bucket-parallelization.md)
Y-Bucket Parallelization is a two-pass parallel algorithm designed for direct volume rendering of irregular or multiple grids, specifically addressing the inefficiencies of ray-casting intersecting cells. It solves the problem of rendering irregular polygonal data by grouping polygons into scanline-based "y-buckets" to maximize spatial coherence and avoid the duplication inherent in tile-based parallelization. Unlike [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) which breaks scanlines, this method parallelizes on a scanline basis to maintain front-to-back visibility order efficiently.

### [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md)
Atomic Fragment List Insertion is a GPU-side algorithm for building a linked list of fragments per pixel using atomic operations to handle concurrent writes from multiple fragments without locks. It solves the problem of accumulating arbitrary numbers of overlapping geometry or volume fragments into a per-pixel data structure, enabling subsequent sorting and compositing passes. It differs from sibling methods like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by deferring the compositing logic to a separate resolve pass, and from [A-buffer linked-list creation and ray processing](3878-a-buffer-linked-list-creation-and-ray-processing.md) by relying on a global atomic counter for list construction rather than geometry shader-based support calculation.

### [Hardware and Platform Specific](2517-hardware-and-platform-specific.md)
Hardware and Platform Specific methods execute volume rendering algorithms directly on the GPU using shader languages (GLSL/HLSL) or compute models (CUDA/OpenCL), bypassing the CPU for the heavy lifting of ray marching and compositing. It is distinguished from sibling methods like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) and [Compute Shader Kernel](1202-compute-shader-kernel.md) by its specific reliance on the graphics pipeline's fragment/vertex shaders or specific vendor APIs (CUDA) rather than generic compute abstractions or tile-based memory optimizations.

### [Parallel Prefix Sum (Scan) Orchestration](2593-parallel-prefix-sum-scan-orchestration.md)
Parallel Prefix Sum (Scan) Orchestration is a GPU-based method for computing prefix sums (scans) of an input array using a multi-timestep iterative reduction strategy with ping-pong buffers. It solves the problem of efficiently aggregating data across parallel threads without race conditions, serving as a foundational primitive for operations like voxel compaction and vertex indexing in volume rendering pipelines. It sets itself apart from sibling methods like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by focusing on the low-level data movement and synchronization required for global aggregation, rather than high-level compositing logic.

### [Multi-core BONO Tree Traversal and Ray Casting with Dynamic Tile Grouping](2801-multi-core-bono-tree-traversal-and-ray-casting-with-dynamic-.md)
A multi-core parallel rendering method that divides screen space into contiguous regions for Phase I (BONO tree traversal to build block lists) and partitions ray casting tasks into Z-ordered tiles grouped by weight for Phase II (dynamic load-balanced ray casting). It sets itself apart from static tiling siblings by using a weight-based Z-order grouping strategy to achieve tight load balancing while minimizing synchronization overhead.

### [Parallel Vector Multiplication](2949-parallel-vector-multiplication.md)
Parallel Vector Multiplication is a fundamental GPU primitive where each thread independently multiplies corresponding elements from two input arrays to produce an output array. It solves the problem of element-wise vector scaling or Hadamard product computation at massive scale by mapping one thread per data element [G]. It is distinct from sibling primitives like [Parallel Prefix Sum](2593-parallel-prefix-sum-scan-orchestration.md) (which requires inter-thread dependency) or [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md) (which requires synchronization), as it involves no communication between threads [G].

### [Parallel Direct Compositing](3201-parallel-direct-compositing.md)
Parallel Direct Compositing is a distributed memory volume rendering technique where image compositing is performed directly across processors following the subdivision of the data domain. It solves the scalability bottleneck of parallel volume rendering by minimizing inter-processor communication overhead during the final image assembly phase. Unlike tiled approaches that require complex synchronization or reduction trees, this method relies on a host processor to collect and combine subimages, accepting a serial bottleneck in display rates in exchange for simplified communication logic and decreasing relative communication costs as processor count increases.

### [CUDA-OpenGL Buffer Interoperability](3209-cuda-opengl-buffer-interoperability.md)
This method enables zero-copy data exchange between CUDA compute kernels and the OpenGL rendering pipeline by mapping OpenGL Buffer Objects (BOs) directly into CUDA memory space. It solves the performance bottleneck of transferring GPU-generated geometry (e.g., isosurface vertices, normals) or particle data through the CPU, allowing CUDA kernels to write directly into buffers readable by OpenGL vertex shaders. Unlike the Unreal Engine backend which uses host-side staging interfaces, this method provides a true zero-copy path for standalone renderers. It is distinct from general CUDA-OpenGL Interoperability [1738-cuda-opl-interoperability.md] by focusing specifically on the lifecycle management of Buffer Objects for vertex/fragment data rather than general resource sharing.

### [Subgroup Operations](3683-subgroup-operations.md)
Subgroup operations are hardware-synchronized collective computations performed across a subgroup (NVIDIA warp, AMD wave) of threads. They solve the problem of high-latency global memory synchronization by allowing threads within a subgroup to exchange data and compute aggregates (sums, ballots) using fast on-chip registers. Unlike atomic operations, which serialize access to global memory, subgroup operations are significantly faster and do not involve global memory synchronization.

### [Atomic Operations for Global Counters](3684-atomic-operations-for-global-counters.md)
Atomic operations for global counters provide a mechanism to safely write to global memory from multiple GPU threads simultaneously, avoiding race conditions (collisions) that occur when concurrent threads attempt to update the same memory location. This method solves the problem of generating unique, thread-safe indices or accumulating values in a shared buffer without requiring coarse-grained locks or serialization. It sets itself apart from sibling techniques like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by operating directly on scalar memory locations rather than managing complex data structures or spatial partitions, and from [Parallel Prefix Sum (Scan) Orchestration](2593-parallel-prefix-sum-scan-orchestration.md) by providing immediate, per-thread unique identifiers rather than aggregated range information.

### [Warp Shuffling](4949-warp-shuffling.md)
Warp Shuffling (specifically Single Buffer Shuffling, SBShuffle) is a warp-level reduction technique that performs parallel reductions by communicating register values directly among threads within a warp, eliminating the need for shared memory allocation. It addresses the memory bandwidth and allocation overhead of shared memory-based reductions by leveraging hardware shuffle instructions available on Kepler architecture and newer GPUs. Unlike shared memory approaches, it allocates a single `float4` buffer from the register space of each thread to store intermediate results.

### [Object Parallel Rendering](5728-object-parallel-rendering.md)
A parallel scheme exploiting parallelism in object space by dividing the volume into p subvolumes assigned to processors.

### [Parallelism and Distribution](6265-parallelism-and-distribution.md)
Parallelism and Distribution addresses the computational bottleneck of volume rendering by distributing ray casting, compositing, or voxel processing tasks across multiple processors, threads, or networked nodes to achieve real-time or near-real-time frame rates [G]. It differs from sibling acceleration techniques like [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) or [Visibility Culling](6226-visibility-culling.md), which optimize the work per ray, by instead increasing the total throughput of the rendering pipeline through hardware or cluster-level concurrency [G].

### [CPU-GPU Memory Transfer and Synchronization](8387-cpu-gpu-memory-transfer-and-synchronization.md)
Memory-mapping the Gaussian buffer on the CPU and copying to host-visible staging buffers for discrete GPUs, which have independent memory connected via PCIe.

### Compute Shader Render Pass
- **step**: [S1, unofficial] Bind `renderPipeline` with descriptor set indices `{0, currentImageIndex}`. [S1, unofficial] Push width and height as `uint32_t[2]` constants. [S1, unofficial] Transition swapchain image layout from `eUndefined` to `eGeneral`. [S1, unofficial] Dispatch compute shader: `(width + 15) / 16` by `(height + 15) / 16` workgroups. [S1, unofficial] Transition image layout to `eColorAttachmentOptimal` if GUI is enabled, else `ePresentSrcKHR`. [S1, unofficial] If GUI enabled, draw ImGui, then transition from `eColorAttachmentOptimal` back to `ePresentSrcKHR`.
- **code**: [S1, unofficial] Render pass dispatch and layout transitions.

### Parallel Reduction
- **definition**: [S2, unofficial] Calculation of vector dot products using NVIDIA's CUDA CUBLAS library. [S2, unofficial] A method to calculate vector dot products using point-wise multiplication followed by a parallel reduction operation within each thread block.
- **step**: [S2, unofficial] Create a dot product handler. [S2, unofficial] Calculate the dot product of arbitrary sized vectors using the `cublasSDot` function with necessary parameters. [S2, unofficial] Write the result into device memory. [S2, unofficial] Point-wise multiplication is performed in the kernel, storing results in shared memory sized to the number of threads in the block. [S2, unofficial] Parallel reduction is performed using a tree-based approach within each thread block to sum the multiplied values.
- **layout**: [S2, unofficial] Shared memory is used to store intermediate results for each variable, with each block maintaining its own copy.
- **complexity**: [S2, unofficial] Complexity of parallel reduction is O(log n), compared to O(n) for traditional summation.

## Background
Parallel execution in volume rendering has evolved from early distributed memory clusters to modern GPU-based architectures. Early methods like [Parallel Direct Compositing](3201-parallel-direct-compositing.md) focused on distributed memory systems with torus networks to scale to thousands of processors [S1, unofficial]. With the advent of GPUs, primitives like [Warp Shuffling](4949-warp-shuffling.md) and [Subgroup Operations](3683-subgroup-operations.md) leveraged hardware-specific features (Kepler+ shuffles, Vulkan subgroups) to reduce latency compared to global memory atomics [S1, unofficial]. Interoperability between compute (CUDA) and graphics (OpenGL/Vulkan) pipelines became critical for interactive rendering, leading to [CUDA-OpenGL Interoperability](1738-cuda-opengl-interoperability.md) and [CUDA-OpenGL Buffer Interoperability](3209-cuda-opengl-buffer-interoperability.md) [S2, unofficial][S4, unofficial][S5][S6, S7, unofficial]. Modern implementations often combine these primitives, such as using [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md) for fragment accumulation followed by [Parallel Prefix Sum](2593-parallel-prefix-sum-scan-orchestration.md) for compaction [S1, unofficial].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
