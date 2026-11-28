# Voxelization

Also called: 3D scan-conversion, grid discretization

## Overview
Voxelization is the process of converting continuous geometric representations, such as triangle meshes, into a discrete 3D occupancy grid or scalar field (voxel array) [S1]. It solves the problem of representing complex surface geometry or volumetric data in a format suitable for volume rendering, ray marching, collision detection, and spatial queries [G]. This family is distinguished from [Procedural and implicit volumes](13-procedural-and-implicit-volumes.md) by producing an explicit, sampled grid rather than an analytic function, and from [Acquisition](14-acquisition.md) by being a computational conversion step rather than a physical sensing process [G].

## Shared principles
All methods in this family map continuous world-space coordinates to discrete integer grid indices $(i, j, k)$ [G]. The core mechanism involves:
1.  **Coordinate Mapping**: Defining a grid resolution (number of cells per axis) and a bounding box (origin and extent) [G]. A world coordinate $x$ is mapped to a grid index $idx$ via:
    $$ idx = \lfloor \frac{x - origin}{voxel\_size} \rfloor $$
    where $voxel\_size$ is the physical dimension of a voxel [S1][S3, unofficial].
2.  **Occupancy Determination**: Evaluating whether a specific voxel cell intersects with, contains, or is influenced by the input primitive [G]. This evaluation varies by method, such as subdivision, ray casting, or binvox algorithms [S1].
3.  **Attribute Assignment**: Storing data in the voxel [G]. This may include occupancy flags, density, or color/normal attributes derived from the source geometry or simulation [S12].

## Subtopics

### [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)
Projects mesh geometry onto a 2D plane and uses hardware/software rasterization to determine voxel coverage. Efficient for surface-only ("shell") voxelization, but may miss voxels intersecting triangle edges without conservative rasterization.

### [Point Cloud Voxelization](1000-point-cloud-voxelization.md)
Maps 3D points directly to grid cells based on coordinate projection, aggregating data within occupied cells. Suitable for unstructured point data where geometric intersection tests are not applicable.

### [VoxelGrid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md)
Fills holes in a surface voxel representation using morphological operations. Chosen when watertight interiors are required from a surface mesh.

### [Voxel Grid Data Generation Pipeline](1208-voxel-grid-data-generation-pipeline.md)
Generates labeled voxel grids from sequential scan data, handling occlusions and temporal consistency by maintaining prior and past grid states. Chosen for dynamic, scan-based environments.

### [Voxel Carving](1213-voxel-carving.md)
Produces a dense, filled voxel grid by intersecting geometric constraints from multiple camera views (silhouettes/depth maps). Explicitly rejects space outside observed bounds, unlike surface-only methods.

### [Coordinate Grid Generation](1273-coordinate-grid-generation.md)
Generates structured spatial coordinates (e.g., in NDC space) aligned with the discrete sampling grid, solving the mapping from indices to continuous space for neural fields or ray casting.

### [Voxel Occupancy Test](1281-voxel-occupancy-test.md)
Queries an existing voxel grid to determine if query points lie within occupied voxels. Does not generate the grid but performs containment checks.

### [3D Scan-conversion](1895-3d-scan-conversion.md)
Converts geometric primitives into a volumetric dataset, combining them with existing volume data using matting operators and rendering via ray casting. Maps directly to 3D volume grid coordinates rather than screen space.

### [Virtual Voxel](1987-virtual-voxel.md)
A rasterization-based renderer for large-scale voxel scenes using virtual textures and geometry, streaming LODs into GPU memory for real-time editing.

### [Voxelization of Lines and Planes](2472-voxelization-of-lines-and-planes.md)
An analytic slab method that guarantees N-separability (no holes) and N-minimality for lines and planes by defining thickness based on normals and resolution.

### [Relaxed Connectivity Voxelization](2481-relaxed-connectivity-voxelization.md)
Ensures topological continuity (6-connected or 26-connected paths) for 1D/2D primitives by intersecting with specific 2D targets within voxels.

### [Mesh-to-Voxel Geometry Shader](2800-mesh-to-voxel-geometry-shader.md)
A real-time DirectX technique emitting cube geometry for grid cells intersecting triangles, allowing immediate lighting calculations.

### [Grid Resolution Calculation](2908-grid-resolution-calculation.md)
Calculates 3D grid resolution based on target voxel count and aspect ratio. Chosen when preserving geometric aspect ratios is critical.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Hardware-accelerated algorithm guaranteeing that every voxel intersecting the model is recognized by computing exact depth ranges per fragment. Chosen for robust collision detection.

### [View-Adaptive Voxelization](3313-view-adaptive-voxelization.md)
Adapts voxel resolution to the viewport for particle simulations, exceeding GPU memory limits via span-based representation.

### [Mesh to Voxel Generation via AABB-Triangle Intersection](3775-mesh-to-voxel-generation-via-aabb-triangle-intersection.md)
Uses SAT-based AABB vs Triangle intersection to determine occupied voxels. Chosen for explicit world-space geometric intersection without screen projection.

### [Voxel Grid Optimization](4131-voxel-grid-optimization.md)
Post-processing techniques (smoothing, compression, structural simplification) to improve rendering performance and memory efficiency.

### [Voronoi-like Segmentation Volume Generation](4218-voronoi-like-segmentation-volume-generation.md)
Generates segmentation volumes by assigning voxels to the nearest neighbor in a Poisson-like point set. Chosen for spatially coherent labeling without input geometry.

### [Sparse and Hierarchical Structures](4391-sparse-and-hierarchical-structures.md)
Uses sparse data structures (e.g., octrees) to represent voxel grids efficiently. Chosen for large-scale scenes or high-resolution data exceeding main memory.

### [Voxel Coloring](4418-voxel-coloring.md)
Uses color consistency across multiple views to distinguish object surface voxels from background. Chosen for 3D shape recovery from multi-view images.

### [Crosshairs Intersection Target Method](4608-crosshairs-intersection-target-method.md)
Uses crosshair targets within voxels to determine intersection for connectivity. Chosen for ensuring topological correctness in path generation.

### [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md)
Voxelize parametric surfaces by building a point boodle and processing in the vertex shader. Chosen for handling complex implicit or free-form surfaces directly.

### [Volumetric Attribute and Density Formation](6304-volumetric-attribute-and-density-formation.md)
Forms density and attribute fields during voxelization. Chosen when semantic content or statistical aggregation is required beyond binary occupancy.

### [Mesh Voxelization](6514-mesh-voxelization.md)
Converts a continuous triangular surface mesh into a discrete 3D voxel grid by determining occupancy via ray-mesh intersection tests. Foundational for solid or surface-only representations.

### [GPU and Parallel Acceleration Voxelization](6878-gpu-and-parallel-acceleration-voxelization.md)
Accelerates conversion of continuous primitives into discrete voxel grids by leveraging GPU parallelism. Chosen for high-performance, real-time applications.

### [Voxel Format Generation](6914-voxel-format-generation.md)
Converts 3D Gaussian Splats into a sparse voxel octree for collision detection, outputting metadata and binary data.

### [3D Gaussian Voxelization](7020-3d-gaussian-voxelization.md)
Converts a set of 3D Gaussians into a set of unique voxels and their associated features for efficient processing by a 3D network.

### [Specific Data Formats and Primitives](7148-specific-data-formats-and-primitives.md)
Handles conversion of specific data formats and primitives into voxel grids.

### [Solid Volume Accumulation via Sphere-Box Intersection](7681-solid-volume-accumulation-via-sphere-box-intersection.md)
Iterates over points and their bounding voxels, accumulating the sphere-box intersection volume into a dense array.

### [Analytic and Algebraic Methods](7913-analytic-and-algebraic-methods.md)
Converts continuous geometric primitives into discrete volumetric grids by evaluating exact geometric intersections or integrals against voxel cells. Chosen for high-fidelity, differentiable representations without rasterization aliasing.

### [Template-based Voxelization](7964-template-based-voxelization.md)
Computes the digitization of a triangle or mesh by determining if a voxel belongs to the digitization based on whether a template centered at the voxel intersects the triangle.

### [Mesh Topology and Robust Digitization](8496-mesh-topology-and-robust-digitization.md)
Focuses on maintaining topological correctness and robustness during the digitization of mesh surfaces into voxels.

## Background
Voxelization originated as a method for 3D scan-conversion, converting geometric models into volume buffers for visualization and rendering [G]. Early work focused on binary voxelization for volume graphics, where models were scan-converted into voxels with pre-computed attributes [G]. The field has expanded to include GPU-accelerated techniques for real-time global illumination (Voxel Cone Tracing) [S2, unofficial], obstacle voxelization for real-time fluid simulation [S6, unofficial], and analytic methods for topological correctness [S7]. Modern applications include coordinate grid generation for neural fields [S2, S4, S5, unofficial] and sparse voxel octrees for large-scale scene representation [S4, unofficial].

## Sources
[S1] trimesh.voxel.creation - trimesh 5.1.1 documentation, https://trimesh.org/trimesh.voxel.creation.html, fetched 2026-10-26
[S2] Friduric/voxel-cone-tracing, https://github.com/Friduric/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
[S3] ZJU-Robotics-Lab/Voxelization_API, https://github.com/ZJU-Robotics-Lab/Voxelization_API, fetched 2026-10-27 (unofficial)
[S4] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S5] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S6] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S7] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
[S8] an1pse/voxel-traversal, https://github.com/an1pse/voxel-traversal, fetched 2026-10-27 (unofficial)
[S9] Point-Tessellated Voxelization, http://www.cs.columbia.edu/cg/raymond/gi12/paper100.pdf, fetched 2026-10-26 (unofficial)
[S10] #AltDevBlog » Implementing Voxel Cone Tracing, https://jahej.com/alt/2013_01_31_implementing-voxel-cone-tracing.html, fetched 2026-10-28 (unofficial)
[S11] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
[S12] Volume visualization, https://dl.acm.org/doi/pdf/10.1145/234313.234383, fetched 2026-11-01
