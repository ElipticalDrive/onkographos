# GPU and Parallel Acceleration Voxelization

Also called: GPU Voxelization, Parallel Voxelization

## Overview
This family encompasses methods that accelerate the conversion of geometric primitives (meshes, points, curves) into voxel grids by leveraging parallel processing architectures, primarily GPUs via CUDA or OpenGL/DirectX, and multi-threaded CPUs. It solves the performance bottleneck of naive CPU-based voxelization for high-resolution grids or complex meshes. It sets itself apart from sequential methods by utilizing massive parallelism (thread-per-voxel, thread-per-primitive, or rasterization-based parallelism) to compute occupancy, attributes, or dual vertices simultaneously [S1, S2, S3, unofficial].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md)
Constructs a linked list of fragment depths per screen pixel using atomic counters and shader storage buffers, allowing for subsequent sorting and processing of depth-sorted voxels. It solves the problem of efficiently capturing multiple geometry intersections per pixel (occlusion/overlapping surfaces) into a structure that can be traversed to determine voxel occupancy, distinguishing itself from single-depth methods like MSAA Voxelization by preserving the full intersection history per pixel.

### [Vessel Voxelizer](2321-vessel-voxelizer.md)
A CUDA-accelerated method that converts vascular structures, defined as line segments with associated radii, into a 3D "fuzzy" voxel volume where each voxel's value represents the fraction of its volume occupied by the vessels. It differs from binary voxelization methods by outputting continuous occupancy values via supersampling, and from general mesh voxelization by operating directly on parametric tube primitives (line segments) rather than polygonal surfaces.

### [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md)
Determines the set of active sparse voxels intersected by a triangle mesh and solves an enhanced Quadratic Error Function (QEF) to compute dual vertex positions within those voxels. It differs from its sibling [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md) by explicitly reconstructing surface geometry via dual vertices rather than just rasterizing occupancy, and from [O-Voxel](3835-o-voxel.md) by serving as the foundational voxelization stage that O-Voxel builds upon for material attribute storage. It solves the problem of converting watertight or non-watertight meshes into a sparse, high-fidelity 3D representation suitable for differentiable rendering or neural reconstruction.

### [Atomic Color Averaging in Voxelization](3529-atomic-color-averaging-in-voxelization.md)
A GPU-based technique for compositing fragment colors into a voxel grid when multiple primitives overlap the same voxel cell. It solves the write-ordering problem inherent in parallel rasterization, where fragments from different triangles arrive at the same destination voxel in arbitrary order, which would otherwise cause flickering and non-time-coherent results during dynamic revoxelization. It sets itself apart from sibling methods like [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md) by avoiding dynamic memory allocation and linked lists, instead using fixed-size integer textures and compare-and-swap loops to maintain a running average directly in the voxel texture.

### [CUDA Voxelizer GPU Voxelization](6757-cuda-voxelizer-gpu-voxelization.md)
A GPU-accelerated method to convert polygon meshes into voxel grids using CUDA, supporting both surface voxelization (faces) and solid voxelization (filled volume). It differs from sibling methods by utilizing a bit-packed voxel table and conservative rasterization techniques adapted for parallel atomic operations, often employing persistent threads and radix sorting for sparse data management.

### [Parallel CPU Voxelization](7135-parallel-cpu-voxelization.md)
A multi-threaded CPU-based voxelization method used as a fallback when no CUDA device is found or when forced via the `-cpu` flag. It utilizes Python's `ThreadPoolExecutor` to parallelize the voxelization process across multiple CPU cores. It is chosen when GPU acceleration is unavailable or when the overhead of GPU invocation is not worth it for very small models.

### [MSAA Voxelization](7412-msaa-voxelization.md)
A technique to achieve conservative rasterization on the GPU by leveraging Multi-Sampling Anti-Aliasing (MSAA) to determine voxel occupancy, thereby avoiding the computational overhead of manual primitive edge expansion in the Geometry Shader. It approximates true conservative rasterization by invoking the pixel shader if any subsample within a pixel is covered by a primitive, which simplifies the rendering pipeline and eliminates the need for post-rasterization voxel-primitive intersection tests to remove redundant voxels.

### [Triangle-Parallel Solid Voxelization](9770-triangle-parallel-solid-voxelization.md)
A data-parallel algorithm that maps a set of input triangles to a discrete 3D voxel grid by dedicating one processing thread to each triangle. It solves the problem of determining the interior of a closed surface by flipping the occupancy state of all voxels lying below the triangle's projection along the primary axis. Unlike sibling methods that parallelize over voxel columns or tiles, this approach parallelizes strictly over the input primitives, making it conceptually simple but potentially inefficient for complex meshes due to warp divergence and memory contention.

### [GPU-based Voxel Query via Conditional Histogram Computation](10393-gpu-based-voxel-query-via-conditional-histogram-computation.md)
This method accelerates voxel query in multivariate volumes by computing joint conditional histograms via GPU volume traversal, avoiding the high overhead of spatial hierarchy structures like octrees for data with uniformly high variance. It differs from sibling methods that rely on spatial subdivision or linked lists by pre-computing statistical distributions of attribute pairs conditioned on a transfer function, allowing for rapid query responses without hierarchical navigation.

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
The concept of GPU-accelerated voxelization emerged with the need for real-time fluid simulation and collision detection in 3D graphics. Early implementations utilized the rasterization pipeline to approximate voxel occupancy, leveraging MSAA and conservative rasterization techniques [S1, S3, unofficial]. Parallel algorithms were later developed to handle solid voxelization and sparse data structures more efficiently, often using CUDA for direct memory access and atomic operations [S2, unofficial]. The field has expanded to include specialized methods for vascular structures, dual-grid representations, and attribute-based queries, reflecting the growing complexity of volumetric data in scientific visualization and game engines [S1, S2, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
[S3] Basics of GPU Voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
