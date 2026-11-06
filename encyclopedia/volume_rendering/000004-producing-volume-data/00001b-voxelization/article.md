# Voxelization

Also called: 3D scan-conversion, grid discretization

## Overview
Voxelization is the process of converting continuous geometric representations (such as triangle meshes, point clouds, or parametric surfaces) into a discrete 3D occupancy grid or scalar field (voxel array) [S1]. It solves the problem of representing complex surface geometry or volumetric data in a format suitable for volume rendering, ray marching, collision detection, and spatial queries [G]. This family is distinguished from [Procedural and implicit volumes](13-procedural-and-implicit-volumes.md) by producing an explicit, sampled grid rather than an analytic function, and from [Acquisition](14-acquisition.md) by being a computational conversion step rather than a physical sensing process.

## Shared principles
All methods in this family map continuous world-space coordinates to discrete integer grid indices $(i, j, k)$. The core mechanism involves:
1.  **Coordinate Mapping**: Defining a grid resolution (number of cells per axis) and a bounding box (origin and extent). A world coordinate $x$ is mapped to a grid index $idx$ via:
    $$ idx = \lfloor \frac{x - origin}{voxel\_size} \rfloor $$
    where $voxel\_size$ is the physical dimension of a voxel [S1][S3, unofficial].
2.  **Occupancy Determination**: Evaluating whether a specific voxel cell intersects with, contains, or is influenced by the input primitive. This evaluation varies by method (e.g., rasterization coverage, ray intersection, point projection) [S1][S2, S3, unofficial].
3.  **Attribute Assignment**: Storing data in the voxel. This may be binary occupancy, continuous density, signed distance, or color/normal attributes [S1, S12].

## Subtopics

### [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)
Projects mesh geometry onto a 2D plane and uses hardware/software rasterization to determine voxel coverage. It is efficient for surface-only ("shell") voxelization but may miss voxels intersecting triangle edges without conservative rasterization.

### [Interior Voxelize](995-interior-voxelize.md)
Determines interior voxels by sampling points within grid cells and testing containment against the mesh surface using ray casting or winding numbers. It is robust to complex topology but computationally heavier per voxel than rasterization.

### [Point Cloud Voxelization](1000-point-cloud-voxelization.md)
Maps 3D points directly to grid cells based on coordinate projection, aggregating data within occupied cells. It is suitable for unstructured point data where geometric intersection tests are not applicable.

### [VoxelGrid from Mesh (Ray Method)](1012-voxelgrid-from-mesh-ray-method.md)
Casts rays from voxel centers to determine occupancy or signed distance. It handles arbitrary mesh orientations robustly and can generate solid volumes or SDFs.

### [VoxelGrid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md)
Fills holes in a surface voxel representation using morphological operations.

### [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md)
A GPU technique using atomic counters and shader storage buffers to build linked lists of fragment depths per pixel, allowing sorting and processing of overlapping geometry. It avoids the memory overhead of full A-Buffer textures.

### [Voxel Grid Data Generation Pipeline](1208-voxel-grid-data-generation-pipeline.md)
Generates labeled voxel grids from sequential scan data, handling occlusions and temporal consistency by maintaining prior and past grid states.

### [Voxel Carving](1213-voxel-carving.md)
Produces a dense, filled voxel grid by intersecting geometric constraints from multiple camera views (silhouettes/depth maps). It explicitly rejects space outside observed bounds, unlike surface-only methods.

### [Coordinate Grid Generation](1273-coordinate-grid-generation.md)
Generates structured spatial coordinates (e.g., in NDC space) aligned with the discrete sampling grid, solving the mapping from indices to continuous space for neural fields or ray casting.

### [Voxel Occupancy Test](1281-voxel-occupancy-test.md)
Queries an existing voxel grid to determine if query points lie within occupied voxels. It does not generate the grid but performs containment checks.

### [Shell-based Voxelization](1748-shell-based-voxelization.md)
Marks voxels within a specified radius of any primitive, creating a boundary mask for subsequent operations.

### [3D Scan-conversion](1895-3d-scan-conversion.md)
Converts geometric primitives into a volumetric dataset, combining them with existing volume data using matting operators and rendering via ray casting. It maps directly to 3D volume grid coordinates rather than screen space.

### [Virtual Voxel](1987-virtual-voxel.md)
A rasterization-based renderer for large-scale voxel scenes using virtual textures and geometry, streaming LODs into GPU memory for real-time editing.

### [Vessel Voxelizer](2321-vessel-voxelizer.md)
Converts vascular line segments with radii into a "fuzzy" voxel volume using supersampling to represent fractional occupancy.

### [Voxelization of Lines and Planes](2472-voxelization-of-lines-and-planes.md)
An analytic slab method that guarantees N-separability (no holes) and N-minimality for lines and planes by defining thickness based on normals and resolution.

### [Relaxed Connectivity Voxelization](2481-relaxed-connectivity-voxelization.md)
Ensures topological continuity (6-connected or 26-connected paths) for 1D/2D primitives by intersecting with specific 2D targets within voxels.

### [CSG Model Voxelization](2497-csg-model-voxelization.md)
Converts CSG models into voxel grids by rendering leaf nodes to 2D slicemaps and performing boolean operations via GPU blending.

### [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md)
Converts meshes to sparse voxels and solves a Quadratic Error Function (QEF) to compute dual vertex positions for surface reconstruction.

### [Mesh-to-Voxel Geometry Shader](2800-mesh-to-voxel-geometry-shader.md)
A real-time DirectX technique emitting cube geometry for grid cells intersecting triangles, allowing immediate lighting calculations.

### [Grid Resolution Calculation](2908-grid-resolution-calculation.md)
Calculates 3D grid resolution based on target voxel count and aspect ratio.

### [Velocity Voxelization](3082-velocity-voxelization.md)
Computes obstacle velocity at grid cells containing boundary voxels.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Hardware-accelerated algorithm guaranteeing that every voxel intersecting the model is recognized by computing exact depth ranges per fragment.

### [View-Adaptive Voxelization](3313-view-adaptive-voxelization.md)
Adapts voxel resolution to the viewport for particle simulations, exceeding GPU memory limits via span-based representation.

### [Atomic Color Averaging in Voxelization](3529-atomic-color-averaging-in-voxelization.md)
Averages fragment colors in voxel textures using integer atomic operations.

### [Mesh to Voxel Generation via AABB-Triangle Intersection](3775-mesh-to-voxel-generation-via-aabb-triangle-intersection.md)
Uses SAT-based AABB vs Triangle intersection to determine occupied voxels.

### [Point Sampling from Mesh](3786-point-sampling-from-mesh.md)
Samples points from a mesh for subsequent voxelization.

### [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md)
Computes the signed Euclidean distance to the nearest surface for each voxel, providing sub-voxel precision.

### [Voxel Grid Optimization](4131-voxel-grid-optimization.md)
Post-processing techniques (smoothing, compression, structural simplification) to improve rendering performance and memory efficiency.

### [Voronoi-like Segmentation Volume Generation](4218-voronoi-like-segmentation-volume-generation.md)
Generates segmentation volumes by assigning voxels to the nearest neighbor in a Poisson-like point set.

### [Porous Voxelization](4361-porous-voxelization.md)
Generates porous meshes by randomly discarding voxels based on spatial hashing.

### [Sparse and Hierarchical Structures](4391-sparse-and-hierarchical-structures.md)
Uses sparse data structures (e.g., octrees) to represent voxel grids efficiently.

### [Polygon Mesh Voxelization via Union of Sets](4605-polygon-mesh-voxelization-via-union-of-sets.md)
Computes voxelization by taking the union of voxel sets from individual primitives.

### [Crosshairs Intersection Target Method](4608-crosshairs-intersection-target-method.md)
Uses crosshair targets within voxels to determine intersection for connectivity.

### [Analytic Voxelization](4969-analytic-voxelization.md)
Evaluates exact geometric intersections or integrals against the voxel grid, avoiding rasterization aliasing.

### [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md)
Voxelize parametric surfaces by building a point boodle and processing in the vertex shader.

### [Volumetric Attribute and Density Formation](6304-volumetric-attribute-and-density-formation.md)
Forms density and attribute fields during voxelization.

### [Voxel Volume Rendering](6488-voxel-volume-rendering.md)
Renders voxel volumes using ray marching through 3D textures.

## Background
Voxelization originated as a method for 3D scan-conversion, converting geometric models into volume buffers for visualization and rendering [S12]. Early work focused on binary voxelization for volume graphics, where models were scan-converted into voxels with pre-computed attributes [S12]. The field has expanded to include GPU-accelerated techniques for real-time global illumination (Voxel Cone Tracing) [S2, unofficial], conservative voxelization for collision detection [S1][S6, unofficial], and analytic methods for topological correctness [S7]. Modern applications include neural radiance fields (NeRFs) requiring coordinate grid generation [S1][S2, S4, S5, unofficial] and sparse voxel octrees for large-scale scene representation [S1][S4, unofficial].

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
