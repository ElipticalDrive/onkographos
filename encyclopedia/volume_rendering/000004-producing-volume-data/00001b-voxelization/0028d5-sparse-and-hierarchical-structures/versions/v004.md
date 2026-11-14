# Sparse and Hierarchical Structures


## Overview
Sparse and Hierarchical Structures convert volumetric data into non-uniform, tree-based layouts (typically octrees) to efficiently store and access sparse or multi-resolution data, avoiding the memory overhead of dense uniform grids [G]. This family recursively subdivides space, allowing resolution to adapt to geometric detail or data density, which distinguishes it from sibling methods that fill a fixed uniform grid [12-voxelization.md] or rely on rasterization [11836-rasterization-and-projection.md]. It is chosen when out-of-core processing, neural scene representations, or high-frequency geometry like hair or terrain require adaptive spatial partitioning [G].

## Shared principles

## Subtopics

### [Out-of-Core Sparse Voxel Octree Construction](1689-out-of-core-sparse-voxel-octree-construction.md)
Constructs an SVO from a mesh in a streaming, bottom-up fashion with bounded memory, differing from in-core algorithms by explicitly managing intermediate voxel grids via disk-based partitioning. Choose it for datasets larger than available main memory.

### [Adaptive Voxelization of Unstructured Grids](3246-adaptive-voxelization-of-unstructured-grids.md)
Converts unstructured tetrahedral/hexahedral grids into a hierarchical 3D texture octree with constant-resolution bricks and a pad-and-stencil algorithm, differing from uniform multi-resolution bricks or sparse sampling. Choose it for interactive rendering of out-of-core scientific datasets on standard hardware.

### [Voxel Grid Optimization](4131-voxel-grid-optimization.md)
Reduces computational and memory overhead through grid-level optimizations like resolution calculation, smoothing, and spatial culling, differing from hierarchical tree structures by focusing on pipeline efficiency and active region restriction. Choose it for optimizing traversal and rendering of sparse or high-resolution volumetric data.

### [Sparse Sampling / Sparse Voxel Grid Generation](4588-sparse-sampling-sparse-voxel-grid-generation.md)
Produces a sparse voxel grid by marking every 5th voxel along each axis as occupied, serving as a predictable test pattern for traversal algorithms rather than processing actual geometry. Choose it for testing spatial locality in traversal code.

### [Solid Voxelization via Octree](6830-solid-voxelization-via-octree.md)
Converts 3D line segments into a solid voxel octree using hierarchical subdivision and explicit "solid" occupancy computation, differing from dense voxelization and standard sparse voxelization by focusing on filled volumes for ray intersection testing. Choose it when efficient ray intersection against solid volumes is required.

### [Neural Sparse Voxel Fields (NSVF)](7144-neural-sparse-voxel-fields-nsvf.md)
Models scenes as voxel-bounded implicit functions in a sparse octree, assigning local implicit functions to non-empty voxels to skip empty space, differing from monolithic NeRF by offering ~10x faster inference. Choose it for photorealistic novel view synthesis with efficient sampling.

### [Sparse Voxelization with Hierarchical LoD](7760-sparse-voxelization-with-hierarchical-lod.md)
Processes high-frequency sparse geometry into multiple LoDs, separating geometric/directional data from material properties and using directional histogram fitting, differing from general-purpose voxelizers by handling anisotropic microgeometry. Choose it for rendering fibers, hair, or thin planes.

### [Sparse Voxel Octree Construction](9181-sparse-voxel-octree-construction.md)
Builds an SVO directly from input geometry using GPU hardware rasterization and image load/store interfaces, bypassing intermediate regular grids and scaling to large scenes via synchronization-free thread launching. Choose it for real-time voxelization of dynamic scenes.

### [Sparse Solid Voxelization](9768-sparse-solid-voxelization.md)
Generates a solid voxel representation directly into a sparse hierarchical structure using a bottom-up construction strategy and conservative surface voxelization, differing from sibling methods by constructing the adapted octree up-front in a massively parallel context. Choose it for memory-efficient solid volume generation.

### Terrain Voxelization into Octants
- **Definition**: A geometry stage that voxelizes terrain into a $128^3$ volume composed of half-block octants, used for local light propagation in Minecraft shaders [S1, unofficial].
- **Parameter**: Voxel resolution is defined as half-block octants, where each voxel represents half a block [S1, unofficial].
- **Pitfall**: Redeclaring Iris's `MC_RENDER_STAGE_*` identifiers as `const int` causes invalid GLSL expansion (e.g., `const int 0 = 0`). The solution is to use Iris's predefined macros directly [S1, unofficial].

### Multi-Level Voxel Backend Processing
- **Step**: Iterate over each voxel size in `voxel_sizes`, compute bounding boxes from point min/max values, and call `mean_by_voxel` to aggregate features, colors, and depth confidence into voxels [S2, unofficial]. If `skip_interpolation` is False, pass voxel data through `point_transformer` and `voxel_to_point_interpolation`; otherwise, use raw voxel features directly [S2, unofficial].
- **Parameter**: `skip_interpolation` is a boolean flag to bypass the expensive KNN interpolation step [S2, unofficial].
- **Pitfall**: Empty point sets (`pts_b.numel() == 0`) are handled by appending None or zero-filled tensors to avoid errors, and the backend core runs in fp32 regardless of outer autocast state due to precision sensitivity of voxel hashing and sparse conv [S2, unofficial].

## Background
Sparse and hierarchical voxel structures are utilized in real-time rendering pipelines and point cloud processing frameworks. In real-time graphics, they are employed for local light propagation and shadow mapping, such as voxelizing terrain into a $128^3$ volume for half-block octant calculations [S1, unofficial]. In point cloud and neural rendering backends, they enable multi-level feature aggregation and interpolation across varying voxel resolutions [S2, unofficial].

## Sources
[S1] Hearthlight: sun/moon shadows, voxel local lights, hand fixed, https://github.com/MeYashverma/Hearthlight/pull/1, fetched 2026-10-26 (unofficial)
[S2] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
