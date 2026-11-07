# Trilinear Volume Sampling for Texture Generation

Also called: Trilinear Texture Sampling, Trilinear Volume Sampling

## Overview
Trilinear volume sampling retrieves attribute values (such as color or material properties) from a 3D texture by interpolating between the eight nearest grid points using normalized coordinates [S1, S2, unofficial]. It solves the problem of mapping continuous world-space positions to discrete volumetric data, enabling the generation of smooth attribute volumes from mesh or point cloud inputs [S2, unofficial].

## How it works
**Inputs:**
- `dataTex`: A 3D CUDA texture object storing volume data. The underlying storage type is `ushort` (unsigned 16-bit integer), but the read mode is configured as `cudaReadModeNormalizedFloat`, mapping values to the $[0, 1]$ range [S1, unofficial].
- `coords` / `valid_pos`: A tensor of 3D positions in world space or grid space [S2, unofficial].
- `attr_volume`: The source 3D attribute data (e.g., color, density) to be sampled [S2, unofficial].
- `grid_size`: The dimensions of the target volume grid $(W, H, D)$ [S2, unofficial].
- `aabb`: The axis-aligned bounding box of the volume [S2, unofficial].
- `voxel_size`: The physical size of a single voxel [S2, unofficial].

**Outputs:**
- `attrs`: A tensor containing the interpolated attribute values at the specified coordinates [S2, unofficial].

**Data Structures:**
- `cudaArray *dataArray`: The CUDA array backing the 3D texture `dataTex` [S1, unofficial].
- `texture<ushort, 3, cudaReadModeNormalizedFloat> dataTex`: The texture reference object [S1, unofficial].
- `torch.Tensor attrs`: Initialized as `torch.zeros(texture_size, texture_size, attr_volume.shape[1], device='cuda')` [S2, unofficial].

**Steps:**
1. **Coordinate Normalization (CUDA Context):** Convert world-space or grid-space coordinates $(x, y, z)$ to normalized texture coordinates in the range $[0, 1]$ [S1, unofficial].
2. **Coordinate Transformation (PyTorch Context):** If mapping from world space, transform coordinates to grid space using the bounding box and voxel size:
   $$
   \text{grid\_coords} = \frac{\text{valid\_pos} - \text{aabb}[0]}{\text{voxel\_size}}
   $$
   where `valid_pos` are the world-space positions [S2, unofficial].
3. **Interpolation:** Perform trilinear interpolation [S2, unofficial].
   - In CUDA, call the built-in texture fetch function:
     $$
     \text{value} = \text{tex3D}(\text{dataTex}, x, y, z)
     $$
     where $x, y, z \in [0, 1]$ [S1, unofficial].
   - In PyTorch, use `grid_sample_3d` with `mode='trilinear'` [S2, unofficial]. The input grid tensor is constructed by concatenating a batch dimension of zeros with the coordinates [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def trilinear_sample_cuda_style(dataTex, x_norm, y_norm, z_norm, W, H, D):
    # Map normalized [0,1] to grid indices [0, dim-1]
    x_float = x_norm * (W - 1)
    y_float = y_norm * (H - 1)
    z_float = z_norm * (D - 1)

    x0 = int(math.floor(x_float))
    y0 = int(math.floor(y_float))
    z0 = int(math.floor(z_float))

    x1 = min(x0 + 1, W - 1)
    y1 = min(y0 + 1, H - 1)
    z1 = min(z0 + 1, D - 1)

    wx = x_float - x0
    wy = y_float - y0
    wz = z_float - z0

    # Fetch 8 surrounding values (simulated dataTex access)
    v000 = dataTex[x0, y0, z0]
    v100 = dataTex[x1, y0, z0]
    v010 = dataTex[x0, y1, z0]
    v110 = dataTex[x1, y1, z0]
    v001 = dataTex[x0, y0, z1]
    v101 = dataTex[x1, y0, z1]
    v011 = dataTex[x0, y1, z1]
    v111 = dataTex[x1, y1, z1]

    # Interpolate along X
    v00 = v000 * (1.0 - wx) + v100 * wx
    v01 = v001 * (1.0 - wx) + v101 * wx
    v10 = v010 * (1.0 - wx) + v110 * wx
    v11 = v011 * (1.0 - wx) + v111 * wx

    # Interpolate along Y
    v0 = v00 * (1.0 - wy) + v10 * wy
    v1 = v01 * (1.0 - wy) + v11 * wy

    # Interpolate along Z
    return v0 * (1.0 - wz) + v1 * wz

def trilinear_sample_pytorch(attr_volume, valid_pos, grid_size, aabb, voxel_size):
    # 1. Initialize output tensor
    attrs = torch.zeros(grid_size[0], grid_size[1], attr_volume.shape[1], device='cuda')

    # 2. Transform world coordinates to grid coordinates
    grid_coords = (valid_pos - aabb[0]) / voxel_size

    # 3. Construct grid tensor for grid_sample_3d
    grid_tensor = torch.cat([torch.zeros_like(valid_pos[:, :1]), valid_pos], dim=-1)

    # 4. Sample using trilinear mode
    # Note: grid_sample_3d expects input shape (N, C, H, W, D)
    # The source passes shape explicitly
    attrs = grid_sample_3d(attr_volume, grid_tensor, shape=torch.Size([1, attr_volume.shape[1], *grid_size]), grid=grid_coords.reshape(1, -1, 3), mode='trilinear')
    return attrs
```

**Parameters:**
- `mode`: 'trilinear' [S2, unofficial].
- `cudaReadMode`: `cudaReadModeNormalizedFloat` [S1, unofficial].

**Edge Cases:**
- **Boundary Conditions:** The CUDA texture sampler handles coordinates outside $[0, 1]$ based on the address mode [S1, unofficial].
- **Precision:** Using `ushort` storage with normalized float read mode provides 16-bit precision [S1, unofficial].

## Subtopics

### [GPU Data Layout](2469-gpu-data-layout.md)
Defines how the volume data is organized in memory (e.g., linear, Z-order) before being bound to a texture.

### [ClearView Unified Buffer Construction](2475-clearview-unified-buffer-construction.md)
Describes the buffer management strategy for transferring volume data to the GPU.

### [Volume Texture Setup](4055-volume-texture-setup.md)
Details the creation of the `cudaArray` and binding it to the `texture` object.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as trilinear interpolation involves a fixed number of arithmetic operations [G].
- **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, determined by the texture size [S1, unofficial].
- **Performance:** Hardware-accelerated in CUDA via `tex3D`, which is efficient for random access patterns [S1, unofficial].
- **Strengths:** Smooth interpolation reduces aliasing; hardware acceleration in CUDA [S1, unofficial].
- **Limitations:** 16-bit storage limits precision; requires careful coordinate normalization [S1, S2, unofficial].

## Background
The method originates from CUDA volume rendering techniques, where 3D textures are used to store volumetric data for ray casting [S1, unofficial]. The PyTorch implementation extends this to attribute volume generation from meshes, leveraging `grid_sample_3d` for differentiable sampling [S2, unofficial].

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S2] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
