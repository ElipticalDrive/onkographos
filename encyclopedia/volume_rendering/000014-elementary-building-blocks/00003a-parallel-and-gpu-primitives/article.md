# Parallel and GPU primitives

Also called: GPU acceleration primitives, parallel rendering primitives

## Overview
This family encompasses the low-level algorithms and hardware mechanisms used to execute volume rendering computations on parallel architectures, primarily GPUs and multi-core CPUs [G]. It solves the performance bottleneck of sequential ray casting and compositing by distributing work across thousands of threads [G]. What sets this family apart from sibling methods like [Ray-primitive intersection](769-ray-primitive-intersection.md) and [Ray generation and camera models](770-ray-generation-and-camera-models.md) is its focus on the execution model, memory synchronization, and data parallelism infrastructure rather than the geometric or optical mathematics of the ray itself [G].

## Shared principles
The core mechanism of this family is the mapping of computational tasks (pixels, rays, or data elements) to parallel execution units (threads, warps, or cores) [G]. Common principles include:
1. **Thread Mapping:** Assigning one or more threads to process specific data elements (e.g., one thread per pixel or one thread per voxel sample) [G].
2. **Memory Hierarchy Utilization:** Leveraging global memory for large datasets, shared/local memory for intermediate results (reductions), and registers for per-thread state [G].
3. **Synchronization:** Managing concurrency conflicts using atomic operations, barriers, or spatial partitioning (tiles) to ensure data consistency without excessive serialization [G].
4. **Data Parallelism:** Performing identical operations on different data elements simultaneously, often requiring coalesced memory access patterns to maximize bandwidth [G].

## Subtopics

### [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md)
A rendering strategy that divides the framebuffer into disjoint spatial tiles to enable lock-free, zero-synchronization parallel processing of semi-transparent primitives. It solves the concurrency conflict inherent in alpha compositing by ensuring that each tile is written to by only one thread, eliminating the need for atomics or locks. This approach is distinct from [Parallel Direct Compositing](3201-parallel-direct-compositing.md) which may use global atomic counters, and from [Y-Bucket Parallelization](2108-y-bucket-parallelization.md) which groups pixels by depth rather than spatial locality.

### [Compute Shader Kernel](1202-compute-shader-kernel.md)
A GPU compute shader kernel is a program executed on the GPU that processes data in parallel using thread IDs to map work items to data elements. It serves as a fundamental primitive for general-purpose GPU (GPGPU) computing, such as generating gradient textures or performing initial volume data preprocessing. It differs from higher-level volume rendering techniques by operating at the raw hardware execution level, requiring manual management of thread indexing and memory access patterns.

### [CUDA-OpenGL Interoperability](1738-cuda-opengl-interoperability.md)
CUDA-OpenGL Interoperability is a mechanism that allows CUDA kernels to read from and write to OpenGL resources (textures, buffers) directly in GPU memory, eliminating the need for CPU-mediated data transfers. It solves the bottleneck of copying rendered volume data or simulation results from CUDA device memory back to system memory for display. It sets itself apart from standard interop methods by supporting both read-only access (via mapped arrays/texture objects) and write access (via surface objects or direct buffer pointers), enabling zero-copy rendering pipelines.

### [Y-Bucket Parallelization](2108-y-bucket-parallelization.md)
Y-Bucket Parallelization is a two-pass parallel algorithm for direct volume rendering that processes polygonal representations of isosurfaces or volume slices by grouping them into scanline-based "y-buckets". It solves the problem of efficiently rendering irregular and multiple grids in software by maximizing spatial coherence and avoiding the computational difficulties associated with ray-casting intersecting cells. Unlike tile-based parallelization methods that break scanlines and lose coherence, this method parallelizes on a scanline basis to limit duplication.

### [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md)
Atomic Fragment List Insertion is a GPU-side algorithm for building a per-pixel linked list of fragments using atomic operations to handle concurrent writes from multiple fragments without locks. It solves the problem of accumulating geometry fragments into a dynamic acceleration structure (A-buffer) for subsequent compositing or ray processing. It differs from brute-force ray marching by avoiding empty space evaluation and from [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by using a global atomic counter for list construction rather than tiled accumulation buffers.

### [Hardware and Platform Specific](2517-hardware-and-platform-specific.md)
Hardware and Platform Specific methods execute volume rendering algorithms directly on the GPU using shader languages (GLSL/HLSL) or compute models (CUDA/OpenCL), bypassing the CPU for the heavy lifting of ray marching and compositing. This approach solves the performance bottleneck of CPU-based ray casting by leveraging massive parallelism. It is distinguished from sibling methods like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) and [Compute Shader Kernel](1202-compute-shader-kernel.md) by its specific reliance on the graphics pipeline's fragment/vertex shaders or specific vendor APIs (CUDA) rather than generic compute abstractions or tile-based memory optimizations.

### [Parallel Prefix Sum (Scan) Orchestration](2593-parallel-prefix-sum-scan-orchestration.md)
Parallel Prefix Sum (Scan) Orchestration is a GPU-based method for computing prefix sums (scans) of an input array using a multi-timestep iterative reduction strategy with ping-pong buffers. It solves the problem of efficiently aggregating data across parallel threads without race conditions, serving as a foundational primitive for operations like voxel compaction and vertex indexing in volume rendering pipelines. It sets itself apart from sibling methods like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by focusing on the low-level data movement and synchronization required for global aggregation, rather than high-level compositing logic.

### [Multi-core BONO Tree Traversal and Ray Casting with Dynamic Tile Grouping](2801-multi-core-bono-tree-traversal-and-ray-casting-with-dynamic-.md)
A multi-core parallel rendering method that divides screen space into contiguous regions for Phase I (BONO tree traversal to build block lists) and partitions ray casting tasks into Z-ordered tiles grouped by weight for Phase II (dynamic load-balanced ray casting). It sets itself apart from static tile allocation methods by using dynamic, weight-based grouping of Z-ordered tiles to achieve better load balancing and cache performance while minimizing synchronization overhead.

### [Parallel Vector Multiplication](2949-parallel-vector-multiplication.md)
Parallel Vector Multiplication is a fundamental GPU primitive where each thread independently multiplies corresponding elements from two input arrays to produce an output array. It solves the problem of element-wise vector scaling or Hadamard product computation at massive scale by mapping one thread per data element. It is distinct from sibling primitives like [Parallel Prefix Sum](2593-parallel-prefix-sum-scan-orchestration.md) (which requires inter-thread dependency) or [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md) (which requires synchronization), as it involves no communication between threads.

### [Parallel Direct Compositing](3201-parallel-direct-compositing.md)
A parallel volume rendering technique where image compositing is performed directly across processors in a distributed memory environment, specifically analyzed for block data subdivision on a bidirectional torus topology.

### [CUDA-OpenGL Buffer Interoperability](3209-cuda-opengl-buffer-interoperability.md)
A lifecycle management technique for Buffer Objects (BOs) shared between OpenGL and CUDA. After allocation in OpenGL, the buffer is registered with CUDA. It alternates accessibility for integration (CUDA) or rendering (OpenGL).

### [Subgroup Operations](3683-subgroup-operations.md)
Operations performed across a subgroup (NVIDIA warp, AMD wave), which are hardware-synchronized groups of threads. Unlike atomics, they do not involve global memory synchronization and are faster.

### [Atomic Operations for Global Counters](3684-atomic-operations-for-global-counters.md)
A technique to safely write to global memory from multiple GPU threads simultaneously by avoiding race conditions (collisions) using atomic operations.

### [Warp Shuffling](4949-warp-shuffling.md)
A warp-level reduction technique using warp shuffling operations (Kepler+) to communicate register values, allocating a single float4 buffer from the register space of each thread.

### Compute Shader Render Pass
- **step**: Bind `renderPipeline` with descriptor set indices `{0, currentImageIndex}`. Push width and height as `uint32_t[2]` constants [S1, unofficial]. Transition swapchain image layout from `eUndefined` to `eGeneral` [S1, unofficial]. Dispatch compute shader: `(width + 15) / 16` by `(height + 15) / 16` workgroups [S1, unofficial]. Transition image layout to `eColorAttachmentOptimal` if GUI is enabled, else `ePresentSrcKHR` [S1, unofficial]. If GUI enabled, draw ImGui, then transition from `eColorAttachmentOptimal` back to `ePresentSrcKHR` [S1, unofficial].
- **code**: Render pass dispatch and layout transitions [S1, unofficial].

### Parallel Reduction
- **definition**: Calculation of vector dot products using NVIDIA's CUDA CUBLAS library. A method to calculate vector dot products using point-wise multiplication followed by a parallel reduction operation within each thread block [S2, unofficial].
- **step**: Create a dot product handler. Calculate the dot product of arbitrary sized vectors using the `cublasSDot` function with necessary parameters [S2, unofficial]. Write the result into device memory. Point-wise multiplication is performed in the kernel, storing results in shared memory sized to the number of threads in the block [S2, unofficial]. Parallel reduction is performed using a tree-based approach within each thread block to sum the multiplied values [S2, unofficial].
- **layout**: Shared memory is used to store intermediate results for each variable, with each block maintaining its own copy [S2, unofficial].
- **complexity**: Complexity of parallel reduction is O(log n), compared to O(n) for traditional summation [S2, unofficial].

### [Object Parallel Rendering](5728-object-parallel-rendering.md)
- **definition**: A parallel scheme exploiting parallelism in object space by dividing the volume into p subvolumes assigned to processors [G].
- **step**: Divide the volume into p subvolumes. Assign tasks associated with each subvolume to processors [G]. Parallel render each subvolume to generate p distributed subimages [G]. Merge subimages into the final image via image compositing [G].

## Background
Parallel and GPU primitives have evolved alongside the development of General-Purpose computing on Graphics Processing Units (GPGPU) [G]. Early volume rendering relied on CPU-based ray casting, which was computationally prohibitive for high-resolution volumes [G]. The introduction of programmable shaders (GLSL/HLSL) and compute APIs (CUDA, OpenCL) allowed for massive parallelization of the ray marching and compositing steps [G]. Techniques like tiled compositing and atomic fragment lists emerged to solve specific concurrency issues in alpha blending and fragment accumulation [G]. Object parallel rendering was developed to handle large-scale datasets by distributing volume data across processors, contrasting with screen-space parallelism [G]. Modern implementations leverage hardware-specific features like warp shuffling and subgroup operations for efficient intra-thread-block communication [G].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
