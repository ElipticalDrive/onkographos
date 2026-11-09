# GPU and Parallel Acceleration Voxelization

Also called: GPU Voxelization, Parallel Voxelization, CUDA Voxelization

## Overview
This family encompasses methods that accelerate the conversion of geometric primitives (meshes, point clouds, lines) into discrete volumetric representations by leveraging parallel processing units, primarily GPUs via CUDA or OpenGL/Vulkan, and multi-threaded CPUs. It solves the performance bottleneck of traditional single-threaded voxelization algorithms, enabling real-time or near-real-time voxel generation for complex scenes. What sets this family apart is the exploitation of massive parallelism—either by assigning one thread per primitive (triangle-parallel), one thread per voxel (voxel-parallel), or by utilizing hardware rasterization pipelines (rasterization-based)—to compute occupancy, depth, or attributes simultaneously, often requiring specialized data structures like atomic buffers, sparse hashmaps, or bit-packed tables to handle concurrent writes.

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md)
Per-Pixel Linked List Voxelization is a GPU-based technique that constructs a linked list of fragment depths per pixel using atomic counters and shader storage buffers, allowing for subsequent sorting and processing of depth-sorted voxels. It solves the problem of variable-depth fragment accumulation in rasterization by deferring the resolution of depth conflicts to a post-processing sort phase, enabling accurate solid voxelization without fixed-depth buffers. It differs from [MSAA Voxelization](7412-msaa-voxelization.md) by using explicit linked lists and atomic operations rather than multisample coverage masks, and from [Atomic Color Averaging in Voxelization](3529-atomic-color-averaging-in-voxelization.md) by preserving individual fragment depths for sorting rather than averaging them immediately.

### [Vessel Voxelizer](2321-vessel-voxelizer.md)
Vessel Voxelizer is a CUDA-accelerated method that converts vascular structures, defined as line segments with associated radii, into a 3D "fuzzy" voxel volume where each voxel's value represents the fraction of its volume occupied by the vessels. It differs from binary voxelization methods by outputting continuous occupancy values via supersampling, and from general mesh voxelization by operating directly on parametric tube primitives (line segments) rather than polygonal surfaces.

### [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md)
Flexible Dual Grid Mesh Voxelization maps a triangle mesh into a sparse voxel representation by determining active voxel occupancy and solving a Quadratic Error Function (QEF) to compute dual vertex positions within those voxels. It differs from its sibling [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md) by explicitly constructing a dual-grid topology (vertices and edges) rather than just rasterizing depth, and from [Triangle-Parallel Solid Voxelization](9770-triangle-parallel-solid-voxelization.md) by focusing on the QEF-based surface approximation suitable for non-manifold geometry rather than binary solid occupancy. It serves as the foundational geometry stage for [O-Voxel](3835-o-voxel.md).

### [Atomic Color Averaging in Voxelization](3529-atomic-color-averaging-in-voxelization.md)
Atomic Color Averaging is a GPU-based technique for compositing multiple fragment colors into a single voxel during parallel voxelization, ensuring time-coherent results despite arbitrary write ordering. It solves the problem of race conditions when multiple triangles rasterize into the same voxel by using integer atomic compare-and-swap operations to incrementally compute the average color, avoiding the need for floating-point atomics which are often unavailable or restricted in standard OpenGL/Vulkan image access. Unlike [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md) which defers compositing, or [CUDA Voxelizer GPU Voxelization](6757-cuda-voxelizer-gpu-voxelization.md) which may use native float atomics, this method relies on a packed integer format and a retry loop to maintain correctness.

### [CUDA Voxelizer GPU Voxelization](6757-cuda-voxelizer-gpu-voxelization.md)
A GPU-accelerated method to convert polygon meshes into voxel grids using CUDA, supporting both surface voxelization (faces) and solid voxelization (filled volume). It differs from sibling methods by utilizing a bit-packed voxel table and conservative rasterization techniques adapted for parallel atomic operations, specifically addressing the performance bottlenecks of atomic contention on older GPU architectures.

### [Parallel CPU Voxelization](7135-parallel-cpu-voxelization.md)
Parallel CPU Voxelization is a multi-threaded method for converting polygonal meshes or point clouds into a discrete volumetric representation, serving as a fallback when GPU acceleration (CUDA) is unavailable or explicitly disabled. It differs from sibling methods by leveraging Python's `ThreadPoolExecutor` to distribute voxel computation across available CPU cores, contrasting with the single-threaded or GPU-specific approaches of its alternatives.

### [MSAA Voxelization](7412-msaa-voxelization.md)
MSAA Voxelization is a GPU-based technique that approximates conservative rasterization by leveraging Multi-Sampling Anti-Aliasing (MSAA) coverage tests to determine voxel inclusion, thereby avoiding the computational overhead of explicit primitive edge expansion in the Geometry Shader. It differs from sibling methods like [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md) by relying on hardware rasterizer coverage rather than explicit sorting or linked lists, and from [Triangle-Parallel Solid Voxelization](9770-triangle-parallel-solid-voxelization.md) by using a fragment-shader-driven approach rather than triangle-centric parallel processing.

### [Triangle-Parallel Solid Voxelization](9770-triangle-parallel-solid-voxelization.md)
Triangle-Parallel Solid Voxelization is a data-parallel algorithm that maps a triangle mesh into a solid volumetric representation by dedicating one thread per triangle to determine which voxels lie inside the object. It solves the problem of converting boundary representations into closed, watertight volume data suitable for ray casting or marching cubes by flipping the inside/outside state of voxels along the projection axis. It differs from tile-based methods by parallelizing directly over the triangle count rather than spatial tiles, which simplifies the work distribution but introduces significant warp divergence and memory contention issues.

### [GPU-based Voxel Query via Conditional Histogram Computation](10393-gpu-based-voxel-query-via-conditional-histogram-computation.md)
This method accelerates voxel queries in multivariate volumes by computing joint conditional histograms via GPU volume traversal, avoiding the high memory overhead of spatial hierarchy structures like octrees for data with uniformly high variance. It differs from sibling methods that rely on spatial partitioning or linked lists by projecting multivariate attribute distributions into 2D histogram bins conditioned on a transfer function, enabling efficient statistical exploration of the volume data.

### Low-Detail Obstacle Voxelization
- **Definition**: Using simplified meshes for obstacles to allow voxelization at every frame with little performance cost [S1, unofficial].
- **Parameter**: Mesh LOD: Use a low level of detail mesh for each obstacle [S1, unofficial].

### Voxelization Engine Workload Distribution
- **Definition**: A pipeline architecture distributing voxelization tasks between CPU and GPU to handle ROI computation, classification, and rasterization efficiently [S2, unofficial].
- **Step**: CPU: Compute Region of Interest (ROI) by updating index buffer dynamically. Perform normal-based classification and bucket sorting for polygonal models. Transfer updated data through AGP port. GPU: Rasterize classified primitives in streaming mode. Encode and store results as 2D textures in three directional sheet buffers. Create proxy rectangles to texture map patches for rendering/access [S2, unofficial].

### Crack-Free Voxelization via Axis Swizzling
- **Definition**: A technique to fix 'cracks' (holes/slots) in voxelization caused by steep primitive slopes relative to the view direction. It selects the projection direction (X, Y, or Z axis) based on the face normal to present the largest face area, avoiding high depth gradients [S3, unofficial].
- **Step**: In the Geometry Shader (GS), per primitive, select the projection direction from the X, Y, and Z axes according to the face normal. Swizzle the X, Y, and Z components of the vertex positions in the Geometry Shader to change the projection direction. This allows rendering from 3 directions without 3 separate render passes. In the Pixel Shader, restore the coordinate components according to the information passed from the Geometry Shader [S3, unofficial].
- **Formula**: Crack Condition: If $|ddx(depth)| > 1.0$ or $|ddy(depth)| > 1.0$, cracks occur perpendicular to the depth direction. Calculated in the pixel shader [S3, unofficial].

## Background
The need for efficient voxelization arose with the demand for real-time fluid simulation and collision detection in 3D games and scientific visualization. Early methods were CPU-bound and too slow for dynamic scenes. The introduction of GPU computing (CUDA) and advanced graphics APIs (OpenGL/Vulkan) enabled parallel processing of geometric primitives. Key developments include the use of conservative rasterization to approximate solid voxelization [S3, unofficial], the use of atomic operations for concurrent writes [S1, unofficial], and hybrid CPU-GPU pipelines for handling large-scale models [S2, unofficial]. These methods are widely used in real-time fluid simulation [S1, unofficial], medical imaging (vessel voxelization) [S1, unofficial], and collision detection in physics engines [S2, unofficial].

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
[S3] https://developer.nvidia.com/content/basics-gpu-voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
