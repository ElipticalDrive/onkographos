# Rasterization and Projection
## Overview
This family generates voxel data by leveraging graphics hardware pipelines, specifically rasterization and projection, to map continuous geometric sources (meshes, implicit surfaces, parametric surfaces) into a 3D grid [S1, S3, S4, unofficial]. It differs from geometric clipping or pure intersection methods by utilizing fragment or stream output to determine voxel occupancy, offering high performance for real-time applications [S1, S3, S4, unofficial].
## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].
## Subtopics

### [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md)
Converts a mathematical implicit surface definition into a discrete voxel representation by rasterizing 2D rectangles within a Region of Interest (ROI) and evaluating the implicit function in a pixel shader. It differs from triangle-based rasterization methods (e.g., [Rasterization-based Voxelization](11755-rasterization-and-gpu-shaders.md)) by operating directly on analytical functions rather than polygonal approximations, allowing it to handle complex, non-polygonal geometry. Choose this when the source geometry is defined analytically rather than as a mesh. See [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md).

### [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md)
Converts parametric surfaces (such as NURBS or subdivision surfaces) into a voxel grid by rasterizing points in the 2D parameter domain directly to 3D space using the GPU vertex shader. It differs from mesh-based voxelization by avoiding polygon tessellation and from implicit surface voxelization by evaluating the surface function explicitly in the vertex shader rather than via ray marching or distance field approximation. This approach is particularly suited for real-time applications where complex parametric models need rapid conversion to volumetric data for transparent illustration or further processing. See [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md).

### [Rasterization and GPU Shaders](11755-rasterization-and-gpu-shaders.md)
Generates voxel data by leveraging graphics hardware pipelines, specifically rasterization and geometry shaders, to map triangle meshes into a 3D grid. It differs from geometric clipping methods by utilizing fragment or stream output to determine voxel occupancy, offering high performance for real-time applications at the cost of potential gaps if conservative rasterization is not used. See [Rasterization and GPU Shaders](11755-rasterization-and-gpu-shaders.md).

### [Mesh Rasterization and Triangle Intersection](11783-mesh-rasterization-and-triangle-intersection.md)
Converts continuous triangular meshes into discrete 3D voxel grids by evaluating the geometric relationship between the mesh primitives and the voxel domain. It solves the problem of digitizing polygonal models for volume rendering or simulation by determining voxel occupancy through explicit intersection tests rather than projection. It differs from [Rasterization-based Voxelization](11755-rasterization-and-gpu-shaders.md) by operating in world space using geometric intersection tests rather than projecting geometry onto screen-space tiles, and from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) which operates on discrete samples rather than continuous geometry. See [Mesh Rasterization and Triangle Intersection](11783-mesh-rasterization-and-triangle-intersection.md).

### Image Space Collision Detection
Transferring intersection computation from geometric space to image space by voxelizing common parts of objects and querying 2D textures [S1, unofficial]. Since voxelization is no longer the bottleneck, the collision detection can reach real-time frame rate even for complex and deformable models. It can handle arbitrarily-shaped geometries and the performance is dependent entirely on the volume resolution [S1, unofficial].

## Background
The technique was proposed for real-time voxelization of complex models, aiming to support applications such as surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial]. The method transfers intersection computation from geometric space to image space, allowing collision detection to reach real-time frame rates [S1, unofficial]. Early implementations were tested on hardware such as the ATI Radeon 9800 Pro [S1, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
