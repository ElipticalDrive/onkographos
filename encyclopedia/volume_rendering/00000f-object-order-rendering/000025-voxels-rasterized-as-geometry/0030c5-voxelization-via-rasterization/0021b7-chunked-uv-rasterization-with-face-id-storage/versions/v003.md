# Chunked UV Rasterization with Face ID Storage

Also called: UV Space Rasterization for Voxelization

## Overview
This method rasterizes a triangular mesh in 2D UV space to generate voxel data, storing the source face ID in the alpha channel of the rasterization buffer. This allows for per-texel face identification and subsequent interpolation of 3D vertex attributes (such as position or normals) from the rasterized face. It differs from [3-axis rasterization voxelization](2480-3-axis-rasterization-voxelization.md) by using a single 2D projection pass per chunk rather than three orthogonal passes, and from [Triangle Projection Voxelization](5432-triangle-projection-voxelization.md) by explicitly managing VRAM usage through face chunking and storing face IDs for attribute reconstruction.

## How it works
**Inputs:**
- `out_vertices`: Tensor of vertex positions, shape $(N, 3)$.
- `out_faces`: Tensor of face indices, shape $(M, 3)$.
- `out_uvs`: Tensor of UV coordinates, shape $(N, 2)$.
- `texture_size`: Integer resolution of the 2D rasterization buffer (e.g., 1024).
- `chunk_size`: Integer number of faces to process per rasterization call (default 100,000).

**Outputs:**
- `rast`: Tensor of shape $(1, \text{texture\_size}, \text{texture\_size}, 4)$ containing rasterization data. The alpha channel stores the face ID offset.
- `pos`: Tensor of interpolated 3D positions for valid pixels.

**Data Structures:**
- `rast`: A 4-channel float32 tensor. Channels 0-2 store interpolated attributes (initially undefined or zero), Channel 3 stores the face ID (or 0 if empty).
- `uvs_rast`: A 3D tensor of shape $(1, N, 3)$ containing homogeneous UV coordinates.

**Steps:**
1. **Prepare UV Coordinates:** Convert 2D UVs to 3D homogeneous coordinates for rasterization. The UVs are scaled to $[-1, 1]$ range, a zero Z-component is added, and a one W-component is added.
   $$ \text{uvs\_rast} = \text{cat}([\text{out\_uvs} \times 2 - 1, \mathbf{0}, \mathbf{1}], \text{dim}=-1). \text{unsqueeze}(0) $$
   Where $\text{cat}$ is concatenation along the last dimension, and $\mathbf{0}, \mathbf{1}$ are tensors of zeros and ones matching the height of `out_uvs`.

2. **Initialize Raster Buffer:** Create a zero-initialized tensor for the raster output.
   $$ \text{rast} = \text{zeros}((1, \text{texture\_size}, \text{texture\_size}, 4), \text{device}=\text{'cuda'}, \text{dtype}=\text{float32}) $$

3. **Chunked Rasterization Loop:** Iterate over faces in chunks of size `chunk_size`.
   For each chunk index $i$ from $0$ to $M$ with step `chunk_size`:
   a. Extract the current chunk of faces: $F_{\text{chunk}} = \text{out\_faces}[i : i + \text{chunk\_size}]$.
   b. Rasterize the chunk into a temporary buffer `rast_chunk`.
      $$ (\text{rast\_chunk}, \_) = \text{dr.rasterize}(\text{ctx}, \text{uvs\_rast}, F_{\text{chunk}}, \text{resolution}=[\text{texture\_size}, \text{texture\_size}]) $$
   c. Compute a validity mask where the alpha channel (face ID) is greater than 0.
      $$ \text{mask\_chunk} = \text{rast\_chunk}[:, :, :, 3:4] > 0 $$
   d. Update the alpha channel of `rast_chunk` by adding the global face index $i$. This encodes the absolute face ID.
      $$ \text{rast\_chunk}[:, :, :, 3:4] \leftarrow \text{rast\_chunk}[:, :, :, 3:4] + i $$
   e. Merge `rast_chunk` into the global `rast` tensor. Pixels already filled by previous chunks (where `mask_chunk` is false) retain their values.
      $$ \text{rast} = \text{where}(\text{mask\_chunk}, \text{rast\_chunk}, \text{rast}) $$

4. **Extract Valid Mask:** Identify all pixels covered by any face.
   $$ \text{mask} = \text{rast}[0, :, :, 3] > 0 $$

5. **Interpolate Attributes:** Use the final `rast` buffer (specifically the face IDs stored in the alpha channel) to interpolate 3D positions from the original vertices.
   $$ \text{pos} = \text{dr.interpolate}(\text{out\_vertices}.\text{unsqueeze}(0), \text{rast}, \text{out\_faces})[0][0] $$

**Parameters:**
- `chunk_size`: 100,000 faces. This limits VRAM usage during the rasterization step [S1, unofficial].

**Pitfalls:**
- **Face ID Overwrite:** The alpha channel is used to store the face ID by adding the loop index $i$. This overwrites any original alpha value from the rasterizer. The mask must be recomputed or stored separately before merging if the original alpha is needed for other purposes [S1, unofficial].
- **Precision:** Ensure the face ID fits within the precision of the alpha channel. For float32, this is generally safe for $M < 2^{24}$, but integer types may be preferred if available in the rasterizer backend.

## Subtopics

### Face Token Layout
A compact encoding for voxel face data within a chunk.
- **Parameters:**
  - `kChunkSize`: 62. The logical size of the voxel chunk [S2, unofficial].
  - `kPaddedSize`: 64. The padded size of the voxel chunk for memory alignment [S2, unofficial].
  - `kPaddedArea`: 4096 ($64 \times 64$). The area of a padded slice [S2, unofficial].
- **Layout:** A `uint64_t` face token encodes a unit face with the following bit fields:
  - Bits 0-5: u coordinate (6 bits).
  - Bits 6-11: v coordinate (6 bits).
  - Bits 12-17: depth (6 bits).
  - Bits 18-20: axis (3 bits).
  - Bit 21: positive direction flag (1 bit).
  - Bits 22-31: material (10 bits).
  *(Note: Source [S2, unofficial] lists bit 20 as positive direction and bits 21-31 as material, totaling 12 bits for material. The sum of bits 0-21 is 22 bits. Bits 22-31 are 10 bits. Total 32 bits used? No, `uint64_t` is 64 bits. The source specifies bits 0-31. The remaining bits 32-63 are not specified.)* [S2, unofficial].

### Face Specification for Padded Coordinates
A struct encoding the orientation of a face within the chunk and its matching padded boundary index.
- **Definition:** A struct containing `fixed_axis`, `fixed_value`, `iter_axes`, and `padded_boundary_index` [S3, unofficial].
- **Formula:** Padded coordinates for a face point $(u, v)$ are calculated as:
  $$ \text{coords}[\text{iter\_axes}[0]] = u + 1 $$
  $$ \text{coords}[\text{iter\_axes}[1]] = v + 1 $$
  $$ \text{coords}[\text{fixed\_axis}] = \text{padded\_boundary\_index} $$
  [S3, unofficial].
- **Layout:**
  - `fixed_axis`: `usize`
  - `fixed_value`: `u8`
  - `iter_axes`: `[usize; 2]`
  - `padded_boundary_index`: `u32`
  [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(M \cdot T)$, where $M$ is the number of faces and $T$ is the texture resolution squared. The chunking adds overhead per chunk but keeps memory bounded.
- **Memory Complexity:** $O(T^2)$ for the raster buffer, independent of $M$, due to chunking.
- **Performance:** Limited by VRAM availability for the raster buffer and the cost of multiple rasterization calls. Suitable for large meshes that do not fit in VRAM for a single pass.
- **Strengths:** Memory efficient for large meshes; enables attribute interpolation.
- **Limitations:** Requires a rasterizer that supports face ID output; UV mapping must be well-defined.

## Background
This method is used in the o-voxel-gpu project for GPU-accelerated voxelization of textured meshes [S1, unofficial]. It leverages standard rasterization pipelines to handle complex geometry and attribute interpolation.

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S3] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
