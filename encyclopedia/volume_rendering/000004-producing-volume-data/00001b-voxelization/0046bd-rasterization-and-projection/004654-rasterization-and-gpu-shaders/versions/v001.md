# Rasterization and GPU Shaders

Also called: GPU-based Voxelization, Geometry Shader Voxelization

## Overview
This family generates voxel data by leveraging graphics hardware pipelines, specifically rasterization and geometry shaders, to map triangle meshes into a 3D grid [S1, unofficial]. It differs from geometric clipping methods by utilizing fragment or stream output to determine voxel occupancy, offering high performance for real-time applications at the cost of potential gaps if conservative rasterization is not used [S1, S3, S4, unofficial].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)
Converts a triangle mesh into a discrete volumetric representation by projecting geometry onto a 3D grid using hardware or software rasterization. It solves the problem of generating surface-level voxel data efficiently, avoiding ray-tracing costs. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by often using sparse hash structures and from [Conservative Voxelization](3247-conservative-voxelization.md) by potentially missing voxels without augmentation. Choose when memory efficiency for sparse meshes is required and real-time performance is critical. [See details](908-rasterization-based-voxelization.md).

### [Mesh-to-Voxel Geometry Shader](2800-mesh-to-voxel-geometry-shader.md)
A real-time DirectX technique that converts mesh objects into voxel-like cubes by iterating over integer grid positions within a triangle's bounding box and emitting explicit cube geometry for cells close to the triangle plane. Unlike rasterization-based voxelization which fills pixels, this method explicitly generates the 3D geometry of every intersecting voxel, resulting in significantly higher vertex and triangle counts. Choose when explicit voxel mesh geometry is required for further GPU processing. [See details](2800-mesh-to-voxel-geometry-shader.md).

### Instanced Voxelization
A technique to draw all voxelization slices in a single draw call using instancing [S1, unofficial]. It uses the instance ID to specify the target slice, reducing API overhead compared to separate draw calls per slice [S1, unofficial].

## Background
Originated in the context of real-time fluid simulation and rendering, specifically detailed in *GPU Gems 3* (Blythe 2006) [S1, unofficial]. It was developed to handle deforming meshes (e.g., skinned characters) by caching transformed vertices via stream output, avoiding recomputation for each slice [S1, unofficial]. It is used in real-time physics simulations and dynamic voxel-based rendering pipelines.

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
