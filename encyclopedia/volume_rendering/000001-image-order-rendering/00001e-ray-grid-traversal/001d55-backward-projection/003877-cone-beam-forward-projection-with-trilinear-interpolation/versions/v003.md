# Cone Beam Forward Projection with Trilinear Interpolation

Also called: Cone Beam Forward Projection, ConeBeam Projector (General 3D)

## Overview
Computes the projection of a 3D volume onto a 2D detector array for cone-beam geometry by integrating voxel values along explicit source-to-detector segments [S1, unofficial]. It differs from its parent [Backward Projection](1491-backward-projection.md) by performing the forward integral using trilinear interpolation on a bricked volume layout, and from [GPU Backprojection](5843-gpu-backprojection.md) by operating on the forward ray-casting direction rather than distributing values back to the volume.

## How it works
**Inputs:**
- `volume`: 3D volume data. Must be converted to a bricked layout with a 1-voxel zero border on all sides. Standard input is `[B, 1, Z, Y, X]` [S1, unofficial].
- `source_positions`: Tensor of shape `[views, 3]` [S1, unofficial].
- `detector_positions`: Tensor of shape `[views, rows, columns, 3]` [S1, unofficial].
- `volume_shape`: Tuple `(z, y, x)` defining the logical dimensions of the volume [S1, unofficial].
- `n_samples`: Integer, number of samples per ray (e.g., 192) [S1, unofficial].

**Outputs:**
- `projections`: Tensor of shape `[B, 1, views, rows, columns]` [S1, unofficial].

**Data Structures:**
- **Layout**: Defined by fields `X`, `Y`, `Z` (volume dimensions), `plane` (stride for detector plane), and `bricks_x` (number of bricks in X direction) [S1, unofficial].
- **Brick Storage**: Volumes are stored in bricks of $4 \times 4 \times 4$ voxels. The Python wrapper `_brick` converts standard tensors to this layout `[bricks * 64, group]` [S1, unofficial].
- **ConeRay**: Struct containing ray origin (`px, py, pz`), direction (`sx, sy, sz`), entry/exit fractions, weight, and axis information for boundary clipping [S1, unofficial].

**Algorithm Steps:**
1.  **Preprocessing**: Convert the input volume to the bricked layout with a 1-voxel zero border using `_brick` [S1, unofficial].
2.  **Kernel Launch**: Launch a CUDA kernel with grid size derived from the projection shape `(views, rows, columns)` and block size `(8, 16, 1)` [S1, unofficial]. The kernel is templated on the number of channels `C` (1, 2, or 4) [S1, unofficial].
3.  **Thread Indexing**: Each thread handles one detector pixel for one view:
    ```python
    column = blockIdx.x * blockDim.x + threadIdx.x
    row = blockIdx.y * blockDim.y + threadIdx.y
    view = blockIdx.z
    ray_index = ((size_t)view * R + row) * W + column
    ```
    where `R` is rows and `W` is columns [S1, unofficial].
4.  **Initialization**: Initialize accumulator array `acc[C]` to `0.0f` and `weight` to `0.0f` [S1, unofficial].
5.  **Ray Construction**: Construct `ConeRay` from source and detector positions and the volume bounding box. If the ray misses the box, it is invalid. If valid, set `weight = ray.weight` [S1, unofficial].
6.  **Sampling Loop**: Loop `k` from `0` to `n_samples - 1`:
    a. Calculate sample position:
    $$
    \begin{aligned}
    px &= \text{fma}(k \cdot \text{weight}, ray.sx, ray.px) \\
    py &= \text{fma}(k \cdot \text{weight}, ray.sy, ray.py) \\
    pz &= \text{fma}(k \cdot \text{weight}, ray.sz, ray.pz)
    \end{aligned}
    $$
    b. Compute integer voxel coordinates:
    $$
    ix = \lfloor px \rfloor, \quad iy = \lfloor py \rfloor, \quad iz = \lfloor pz \rfloor
    $$
    c. If voxel coordinates changed from previous step (`ix != x0 || iy != y0 || iz != z0`):
    i. Compute cell index and fractional offsets `ox, oy, oz` using a cell function [S1, unofficial].
    ii. Fetch 8 voxel values `v[8][C]` from the bricked volume storage [S1, unofficial].
    d. Compute fractional offsets for interpolation:
    $$
    dx = px - ix, \quad dy = py - iy, \quad dz = pz - iz
    $$
    e. Perform trilinear interpolation:
    - For `C=1`:
    $$
    acc[0] += \text{trilinear}(v, dx, dy, dz)
    $$
    - For `C>1`:
    $$
    \begin{aligned}
    wx &= [1-dx, dx] \\
    wy &= [1-dy, dy] \\
    wz &= [1-dz, dz]
    \end{aligned}
    $$
    Loop `q` from 0 to 7:
    $$
    w = (wz[q >> 2] \cdot wy[(q >> 1) \& 1]) \cdot wx[q \& 1]
    $$
    $$
    acc[c] += w \cdot v[q][c] \quad \text{for all } c \in [0, C-1]
    $$
7.  **Accumulation**: After the loop, write results:
    $$
    out[(size_t)c \cdot plane + ray\_index] = acc[c] \cdot weight
    $$
    for `c < nb` (number of output channels) [S1, unofficial].

**Pitfalls:**
- Memory access is guarded by clamping in the cell function: `x = min(max(x0, 0), l.X)`, etc., to prevent out-of-bounds access [S1, unofficial].
- The volume must have a 1-voxel zero border; otherwise, boundary samples may read uninitialized or incorrect data [S1, unofficial].

## Subtopics

### [Differentiable Forward Projection (Mesh)](13343-differentiable-forward-projection-mesh.md)
This method computes the differentiable forward projection of 3D triangle meshes onto 2D detector arrays for both parallel and cone-beam geometries. It differs from volume-based approaches (like its parent [Cone Beam Forward Projection with Trilinear Interpolation](8247-cone-beam-forward-projection-with-trilinear-interpolation.md)) by operating directly on mesh topology (vertices and faces) and utilizing a custom CUDA rasterizer to integrate attenuation coefficients along rays, enabling gradient backpropagation through the projection process.

### Image Scanning for Tomography
A method to generate tomographic projections (scans) from a source image by integrating pixel values along lines at various angles and offsets [S2, unofficial].
- **Steps**:
    1. Calculate sampling steps: `y_step = 2.0 / base_width`, `x_step = 2.0 / base_height`.
    2. Calculate pixel center offsets: `y_offset = (image.height - 1.0) / 2.0`, `x_offset = (image.width - 1.0) / 2.0`.
    3. Iterate over image pixels: for `y_idx` in `0..height`, for `x_idx` in `0..width`:
       $$
       \begin{aligned}
       y &= (y\_idx - y\_offset) \cdot y\_step \\
       x &= (x\_idx - x\_offset) \cdot x\_step
       \end{aligned}
       $$
       Call `value_fn(x, y, pixel_value)` [S2, unofficial].
- **Parameters**: `rays` (number of projection rays, typically 35), `angles` (number of projection angles, typically 40) [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of voxels sampled per ray and the number of rays.
- **Memory Complexity**: Linear with respect to the volume size (bricked layout) and output projection size.
- **Performance**: On an RTX 2080 Ti with a $256^3$ phantom, 360 views, and $384 \times 384$ detectors: Forward projection takes 0.26s, Adjoint 0.33s, Forward with geometry gradient 0.40s [S1, unofficial]. The PyTorch path takes 207s (forward) and 232s (adjoint), and runs out of memory for larger scenes [S1, unofficial].
- **Strengths**: The CUDA backend uses runtime-compiled kernels for float32, storing nothing per sample, and is significantly faster than the PyTorch path [S1, unofficial].
- **Limitations**: The PyTorch path retains interpolation stencils for autograd but fits only small scans [S1, unofficial].

## Background
Developed as part of the TorchTomo library by the BIAI Lab, this method is part of a differentiable tomography framework supporting parallel, fan, and cone-beam geometries [S1, unofficial]. It provides exact discrete adjoints for use in gradient-based optimization and reconstruction algorithms [S1, unofficial].

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S2] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
