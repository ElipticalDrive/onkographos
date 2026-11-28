# Chunked UV Rasterization with Face ID Storage

Also called: UV Space Rasterization for Voxelization

## Overview
This method rasterizes a 3D mesh in 2D UV space to generate voxel data, using memory-friendly chunking to handle large geometries. It differs from [3-axis rasterization voxelization](2480-3-axis-rasterization-voxelization.md) by projecting onto a single UV plane per pass and storing face identifiers to enable precise 3D position interpolation, rather than relying on depth buffers or multiple axis projections. It addresses the VRAM limitations of full-mesh rasterization by processing faces in batches.

## How it works
**Inputs:**
- `out_vertices`: Tensor of vertex positions, shape $(N_v, 3)$.
- `out_faces`: Tensor of face indices, shape $(N_f, 3)$.
- `out_uvs`: Tensor of UV coordinates, shape $(N_v, 2)$.
- `texture_size`: Integer resolution of the 2D rasterization buffer (e.g., 1024).
- `chunk_size`: Integer number of faces to process per rasterization call (typically 100,000).

**Outputs:**
- `voxel_positions`: Tensor of 3D positions for valid voxels.
- `voxel_mask`: Boolean mask indicating valid voxel locations.

**Data Structures:**
- `rast`: A 4D tensor of shape $(1, \text{texture\_size}, \text{texture\_size}, 4)$, initialized to zeros. The first three channels store interpolated UV or depth values (implementation dependent), and the fourth channel (alpha) stores the accumulated face ID.
- `uvs_rast`: A 3D tensor of shape $(1, N_v, 4)$, constructed by padding UVs into homogeneous coordinates for the rasterizer.

**Steps:**

1.  **Prepare UV Coordinates:**
    Transform the input UVs into the rasterizer's expected range $[-1, 1]$ and pad them.
    $$
    \text{uvs\_rast} = \text{cat}([\text{out\_uvs} \times 2 - 1, \mathbf{0}, \mathbf{1}], \text{dim}=-1).unsqueeze(0)
    $$
    Where $\mathbf{0}$ and $\mathbf{1}$ are tensors of zeros and ones matching the vertex count, and `cat` concatenates along the last dimension.

2.  **Initialize Raster Buffer:**
    Create a zero-initialized tensor on the target device (e.g., CUDA):
    $$
    \text{rast} = \text{zeros}((1, \text{texture\_size}, \text{texture\_size}, 4))
    $$

3.  **Chunked Rasterization Loop:**
    Iterate over the faces in chunks of size $C$ (e.g., 100,000). Let $i$ be the starting index of the current chunk.
    
    a. **Rasterize Chunk:**
    Call the 2D rasterizer with the prepared UVs and the current face chunk `out_faces[i : i+C]`.
    $$
    (\text{rast\_chunk}, \_) = \text{dr.rasterize}(\text{ctx}, \text{uvs\_rast}, \text{out\_faces}[i:i+C], \text{resolution}=[\text{texture\_size}, \text{texture\_size}])
    $$
    
    b. **Compute Mask:**
    Identify pixels covered by the current chunk.
    $$
    \text{mask\_chunk} = \text{rast\_chunk}[..., 3:4] > 0
    $$
    *Note: In the first iteration, this checks if the buffer is empty. In subsequent iterations, it checks if the current chunk covers pixels that might have been covered by previous chunks, depending on the specific rasterizer's overwrite behavior. The source implementation uses `torch.where` to merge.*
    
    c. **Store Face ID:**
    The face ID is stored in the alpha channel. To distinguish faces from different chunks, the loop index $i$ is added to the alpha channel of the raster output.
    $$
    \text{rast\_chunk}[..., 3:4] \leftarrow \text{rast\_chunk}[..., 3:4] + i
    $$
    
    d. **Merge Results:**
    Update the global raster buffer. Pixels covered by the current chunk are overwritten; others retain their previous values.
    $$
    \text{rast} = \text{torch.where}(\text{mask\_chunk}, \text{rast\_chunk}, \text{rast})
    $$

4.  **Extract Valid Voxels:**
    Identify all pixels that were covered by any face.
    $$
    \text{mask} = \text{rast}[0, ..., 3] > 0
    $$

5.  **Interpolate 3D Positions:**
    Use the accumulated face IDs in the alpha channel to look up the corresponding triangle and interpolate the 3D vertex positions at the pixel centers.
    $$
    \text{pos} = \text{dr.interpolate}(\text{out\_vertices}.unsqueeze(0), \text{rast}, \text{out\_faces})[0][0]
    $$
    The resulting `pos` tensor contains the 3D coordinates for each valid voxel.

**Parameters:**
- `chunk_size`: 100,000 faces. This value is chosen to limit VRAM usage during the rasterization step [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Face ID Overwrite:** The face ID is stored in the alpha channel by adding the loop index $i$. This overwrites the original alpha value produced by the rasterizer. The mask must be recomputed or stored separately before merging to ensure correct coverage detection [S1, unofficial].
- **Precision:** Using floating-point tensors for the raster buffer allows for interpolation but requires careful handling of the integer face ID storage in the alpha channel.

## Subtopics

### Face Token Layout
A compact encoding for voxel face data used in chunked voxelization systems. It packs face orientation, position, and material into a single 64-bit integer.
- **Bit Layout:**
    - Bits 0-5: $u$ coordinate (6 bits).
    - Bits 6-11: $v$ coordinate (6 bits).
    - Bits 12-17: Depth (6 bits).
    - Bits 18-20: Axis (3 bits).
    - Bit 21: Positive direction flag (1 bit).
    - Bits 22-31: Material ID (10 bits).
- **Parameters:**
    - `kChunkSize`: 62 (logical chunk dimension).
    - `kPaddedSize`: 64 (padded chunk dimension for alignment).
    - `kPaddedArea`: 4096 ($64 \times 64$) [S2, unofficial].

### Face Specification for Padded Coordinates
A structure defining the orientation and boundary mapping of a face within a padded voxel chunk.
- **Definition:** Encodes the fixed axis, fixed value, iteration axes, and boundary index for a face point.
- **Fields:**
    - `fixed_axis`: Index of the axis perpendicular to the face.
    - `fixed_value`: Coordinate value along the fixed axis.
    - `iter_axes`: Array of two indices representing the in-plane axes.
    - `padded_boundary_index`: The index in the padded array corresponding to the fixed axis position.
- **Formula:** Padded coordinates for a face point $(u, v)$ are calculated as:
    $$
    \begin{aligned}
    \text{coords}[\text{iter\_axes}[0]] &= u + 1 \\
    \text{coords}[\text{iter\_axes}[1]] &= v + 1 \\
    \text{coords}[\text{fixed\_axis}] &= \text{padded\_boundary\_index}
    \end{aligned}
    $$
    The $+1$ offset accounts for boundary padding [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of faces $N_f$ and the number of chunks $N_c = \lceil N_f / \text{chunk\_size} \rceil$. Each chunk requires a full 2D rasterization pass.
- **Memory Complexity:** Dominated by the raster buffer size $O(\text{texture\_size}^2)$ and the vertex/face storage. Chunking prevents VRAM overflow for large meshes.
- **Strengths:** Memory efficient for large scenes; leverages highly optimized 2D rasterizers; accurate position interpolation via face IDs.
- **Limitations:** Requires a differentiable or compatible 2D rasterizer; UV mapping quality affects voxel accuracy; single-pass projection may suffer from aliasing if not resolved.

## Background
This method is implemented in the `o-voxel-gpu` project, which provides GPU-accelerated voxelization tools. It utilizes PyTorch and custom CUDA extensions for rasterization and interpolation. The approach is part of a broader trend in differentiable rendering and voxel-based neural radiance fields (NeRF) where mesh-to-voxel conversion is required for volume rendering [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S3] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
