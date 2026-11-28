# Position Map and Image-Space Transfer


## Overview
This family of techniques converts 3D geometric data into voxel grids by projecting or transferring information into 2D image space (textures) before reconstructing the 3D volume [G]. Unlike direct 3D rasterization, these methods leverage the GPU's fixed-function rasterizer or image load/store operations on 2D textures to determine voxel occupancy [G]. They are distinguished from direct triangle rasterization by operating on pre-computed surface samples or 2D texture slices, ensuring connectivity or enabling real-time performance insensitive to scene complexity [G].

## Shared principles
All methods in this family rely on the mapping of world-space coordinates to discrete voxel indices [G]. Given a world position $p = (p_x, p_y, p_z)$, a bounding box defined by minimum $p_{min}$ and maximum $p_{max}$, and a volume resolution $D$, the voxel index $i$ is calculated as:

$$
i = D \times \frac{p - p_{min}}{p_{max} - p_{min}}
$$

The result is truncated (floored) to an integer to yield the discrete voxel index [G]. This formula is applied consistently across Position Map Voxelization, Billboard Voxelization, and Hardware-accelerated Surface Voxelization to map continuous geometry to the discrete 3D grid [G].

## Subtopics

### [Position Map Voxelization](2604-position-map-voxelization.md)
A post-processing step that converts a 2D position map (containing world-space coordinates of previously identified surface points) into a 3D voxel grid. It differs from direct triangle rasterization by operating on pre-computed surface samples, ensuring connectivity by marking the primary voxel and its 8 immediate neighbors as active. It uses a full-screen quad to process every sample in the position map, writing directly to a 3D volume texture.

### [Billboard Voxelization](2659-billboard-voxelization.md)
A two-pass rasterization-based technique that initializes a 3D volume by rendering instanced quads (billboards) into a 2D position map framebuffer object (FBO). It differs from [Position Map Voxelization](2604-position-map-voxelization.md) by explicitly generating billboard geometry per voxel to write nearest-neighbor world positions, rather than relying on standard triangle rasterization into a depth buffer. It is suitable for initializing voxel grids from known voxel centers.

### [Hardware-accelerated surface voxelization via 2D textures](4042-hardware-accelerated-surface-voxelization-via-2d-textures.md)
This method transfers the geometric intersection computation from 3D space to 2D image space by rasterizing model surfaces into 2D textures (worksheets). It is distinguished from slicing-based voxelization by rasterizing the scene approximately once rather than $N$ times (where $N$ is the volume resolution), making its performance insensitive to scene complexity and dependent only on volume resolution. It is optimal for real-time collision detection and volumetric rendering of complex models.

## Background
Not given in the sources.

## Sources
