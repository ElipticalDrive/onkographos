# Position Map and Image-Space Transfer

Also called: Image-Space Voxelization, Position Map Transfer

## Overview
Position Map and Image-Space Transfer is a family of rasterization-based voxelization techniques that decouple surface intersection from volume grid population by using intermediate 2D image textures (position maps) to store world-space surface coordinates. It solves the problem of efficiently mapping continuous geometric surfaces to discrete voxel grids by leveraging the GPU's rasterization pipeline to compute visibility and position, rather than performing geometric clipping or ray-marching against the volume grid directly. It differs from direct triangle rasterization into a voxel grid by operating on pre-computed surface samples in image space, and from hardware-accelerated surface voxelization via 2D textures by explicitly using a 2D image store to write directly into a 3D volume texture during a second pass [S1, unofficial].

## Shared principles
The family relies on a two-stage image-space transfer mechanism:

1.  **Position Map Generation:** A 2D texture (the position map) is rendered to, storing world-space coordinates (typically `vec4` with `xyz` as position and `a` as a validity flag) for the nearest surface point at each screen pixel. This is achieved by rasterizing geometry (triangles or instanced quads) with depth testing enabled, ensuring only the closest surface fragment updates the texture. The position map uses high-precision formats (e.g., `rgba32f`) to preserve world-space accuracy [S1, unofficial].
2.  **Voxel Grid Population:** A second pass processes the position map. A full-screen quad is rendered, and for each fragment, the stored world-space position is converted to 3D voxel indices using the volume's bounding box and resolution. The corresponding voxel in the 3D volume texture is marked as active (e.g., set to 1). This pass typically disables depth testing and color masking to allow direct writes to the volume texture without overwriting concerns [S1, unofficial].

The conversion from world position $pos = (pos.x, pos.y, pos.z)$ to voxel indices $(x, y, z)$ for a volume of dimension `voxelDim` and bounds $[xBounds, yBounds, zBounds]$ is:

$$
x = \text{voxelDim} \times \frac{pos.x - xBounds.x}{xBounds.y - xBounds.x}
$$

$$
y = \text{voxelDim} \times \frac{pos.y - yBounds.x}{yBounds.y - yBounds.x}
$$

$$
z = \text{voxelDim} \times \frac{pos.z - zBounds.x}{zBounds.y - zBounds.x}
$$

Standard integer truncation or floor is implied by the return type `ivec3` in typical GLSL implementations [G].

## Subtopics

### [Position Map Voxelization](2604-position-map-voxelization.md)
Position Map Voxelization is a post-processing step that converts a 2D position map into a 3D voxel grid by rasterizing pre-computed surface points into a volume texture. It differs from direct triangle rasterization by operating on surface samples and ensuring connectivity by marking the primary voxel and its 8 immediate neighbors as active. It is distinct from [Billboard Voxelization](2659-billboard-voxelization.md) and [Hardware-accelerated surface voxelization via 2D textures](4042-hardware-accelerated-surface-voxelization-via-2d-textures.md) because it explicitly uses a 2D image store to write directly into a 3D volume texture during a second pass. Choose this method when connectivity and surface continuity are critical, and the position map is already available from a previous pass. See [Position Map Voxelization](2604-position-map-voxelization.md).

### [Billboard Voxelization](2659-billboard-voxelization.md)
Billboard Voxelization is a two-pass technique that initializes a position map by rendering instanced quads (billboards) corresponding to voxel centers, writing the nearest world-space positions to a framebuffer. It differs from [Position Map Voxelization](2604-position-map-voxelization.md) by explicitly rendering geometry to establish depth, and from [Hardware-accelerated surface voxelization via 2D textures](4042-hardware-accelerated-surface-voxelization-via-2d-textures.md) by using a full position map rather than just depth or surface normals [G]. Choose this method when you need to voxelize a sparse or implicit volume by projecting voxel centers onto the surface. See [Billboard Voxelization](2659-billboard-voxelization.md).

### [Hardware-accelerated surface voxelization via 2D textures](4042-hardware-accelerated-surface-voxelization-via-2d-textures.md)
This method transfers intersection computation from geometric space to image space by rasterizing model surfaces into 2D textures, enabling collision detection and volumetric rendering at real-time frame rates. It differs from slicing-based voxelization by rasterizing the model approximately once rather than $N$ times (where $N$ is the volume resolution along $z$), making performance insensitive to scene complexity. Unlike Layered Depth Image (LDI) extraction, it is view-independent and avoids CPU-side sorting of depth lists. Builds on [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md). Choose this method for real-time collision detection and rendering of complex deformable models. See [Hardware-accelerated surface voxelization via 2D textures](4042-hardware-accelerated-surface-voxelization-via-2d-textures.md).

### Box-Basis Rasterization for Volume Lines
#### definition
- [S1, unofficial] A method to map sub-pixel accurate x-coordinate pairs $(x_{f1}, x_{f2})$ per AMR cell to integer pixel bins and compute average intensity values, borrowing from the basis function method but using box-shaped basis functions.
#### step
- [S1, unofficial] Create a set of $W$ bins (one for each pixel in the plot's x-dimension) and initialize their values to 0.
- [S1, unofficial] Project the coordinate pairs $(x_{f1}, x_{f2})$ to integer coordinates in the range $[0, W-1]$ and iterate over these, increasing the overlapping bins by the value of the corresponding AMR cell.
- [S1, unofficial] Maintain a per-bin counter that is incremented whenever a bin value is increased.
- [S1, unofficial] After processing all pairs, iterate over the bins and divide each bin value by its counter to obtain the average intensity value of all AMR cells projecting to that bin.
#### pitfall
- [S1, unofficial] Pairs $(x_{f1}, x_{f2})$ that span multiple bins turn the plot into a step function; many line segments map to single bins, which is accepted for simplicity over generality.

## Background
The concept of using image-space techniques for volume data generation and analysis has evolved alongside GPU rasterization capabilities. The specific application of position maps for voxelization leverages the GPU's ability to render depth-buffered geometry to compute surface intersections efficiently. The box-basis rasterization variant was introduced in the context of visual analysis of large multi-field AMR (Adaptive Mesh Refinement) data on GPUs, where it was used to project cell values onto 1D plots for interactive analysis [S1, unofficial]. This approach borrows from the basis function method by Wald et al. [11], adapting it for AMR data visualization [S1, unofficial].

## Sources
[S1] PDF Visual Analsyis of Large Multi-Field AMR Data on GPUs Using Interactive ..., https://www.cs.bilkent.edu.tr/~gudukbay/publications/papers/conf_papers/Zellmann_Et_Al_IEEE_VIS_2023.pdf, fetched 2026-10-27 (unofficial)
