# Chunked UV Rasterization with Face ID Storage

Also called: UV Space Rasterization for Voxelization

## Overview
This method converts a triangle mesh into a voxelized representation by rasterizing triangles into a 2D texture buffer in UV space, storing the face ID in the alpha channel to enable per-texel face identification and subsequent 3D position interpolation [S1, unofficial]. It builds on [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by decoupling the rasterization step from the 3D grid storage, allowing for memory-friendly chunked processing of large meshes. Unlike [3D Grid Rasterization](903-3d-grid-rasterization.md) which operates directly in 3D, this approach projects the mesh onto a 2D plane, leveraging standard 2D rasterization hardware or libraries to determine coverage and face identity before mapping results back to 3D voxel coordinates.

## How it works
**Inputs:**
- `out_vertices`: Tensor of vertex positions, shape $(N_v, 3)$.
- `out_faces`: Tensor of face indices, shape $(N_f, 3)$.
- `out_uvs`: Tensor of UV coordinates for vertices, shape $(N_v, 2)$.
- `texture_size`: Resolution of the 2D rasterization buffer (e.g., 1024).
- `chunk_size`: Number of faces to process per rasterization call (default 100,000) [S1, unofficial].

**Outputs:**
- `rast`: A 4D tensor of shape $(1, \text{texture\_size}, \text{texture\_size}, 4)$ representing the rasterization buffer. The RGB channels store interpolated UV or depth information (implementation dependent), and the Alpha channel stores the face ID.
- `pos`: Interpolated 3D positions for each valid texel, shape $(\text{texture\_size}, \text{texture\_size}, 3)$.

**Data Structures:**
- `rast`: Float32 tensor. The alpha channel is used to store the global face index $i$.
- `mask`: Boolean tensor indicating valid texels where alpha $> 0$.

**Steps:**

1.  **Prepare UV Coordinates:**
    Convert vertex UVs to clip space coordinates for rasterization. The UVs are scaled to $[-1, 1]$ and extended to homogeneous coordinates $(u, v, 0, 1)$ [S1, unofficial].
    $$
    \text{uvs\_rast} = \text{cat}([\text{out\_uvs} \times 2 - 1, \mathbf{0}, \mathbf{1}], \text{dim}=-1)
    $$
    Shape: $(N_v, 4)$.

2.  **Initialize Rasterization Buffer:**
    Create a zero-initialized tensor for the 2D buffer [S1, unofficial].
    $$
    \text{rast} = \text{zeros}((1, \text{texture\_size}, \text{texture\_size}, 4), \text{device}=\text{'cuda'}, \text{dtype}=\text{float32})
    $$

3.  **Chunked Rasterization Loop:**
    Iterate over faces in chunks of size `chunk_size` (default 100,000) [S1, unofficial]. For each chunk starting at index $i$:
    
    a. **Rasterize Chunk:**
    Call the rasterization function `dr.rasterize` with the UV coordinates and the current face chunk. The function returns a rasterization result `rast_chunk` and auxiliary data [S1, unofficial].
    $$
    \text{rast\_chunk}, \_ = \text{dr.rasterize}(\text{ctx}, \text{uvs\_rast}, \text{out\_faces}[i:i+\text{chunk\_size}], \text{resolution}=[\text{texture\_size}, \text{texture\_size}])
    $$
    
    b. **Compute Mask:**
    Identify texels covered by the current chunk. A texel is valid if its alpha value in `rast_chunk` is greater than 0 [S1, unofficial].
    $$
    \text{mask\_chunk} = \text{rast\_chunk}[..., 3:4] > 0
    $$
    
    c. **Store Face ID:**
    Update the alpha channel of `rast_chunk` with the global face index $i$. This overwrites any previous alpha value, so the mask must be computed *before* this update or stored separately to avoid overwriting valid data from previous chunks if merging logic is complex [S1, unofficial].
    $$
    \text{rast\_chunk}[..., 3:4] += i
    $$
    Note: The source code adds $i$ to the alpha channel. If the alpha channel was previously used for depth or other attributes, this logic assumes the alpha channel is exclusively used for face ID storage in this specific implementation [S1, unofficial].
    
    d. **Merge:**
    Update the global `rast` buffer. Only texels covered by the current chunk are updated [S1, unofficial].
    $$
    \text{rast} = \text{torch.where}(\text{mask\_chunk}, \text{rast\_chunk}, \text{rast})
    $$

4.  **Extract Valid Pixels:**
    After processing all faces, create a global mask of all valid texels in the buffer [S1, unofficial].
    $$
    \text{mask} = \text{rast}[0, ..., 3] > 0
    $$

5.  **Interpolate 3D Positions:**
    Use the rasterization result `rast` (which now contains face IDs) to interpolate the 3D vertex positions for each valid texel [S1, unofficial].
    $$
    \text{pos} = \text{dr.interpolate}(\text{out\_vertices}.unsqueeze(0), \text{rast}, \text{out\_faces})[0][0]
    $$
    The `dr.interpolate` function uses the face IDs stored in the alpha channel of `rast` to look up the corresponding triangle and barycentric coordinates (implicitly stored or derived from the rasterization step) to interpolate the vertex positions.

**Parameters:**
- `chunk_size`: 100,000 faces per rasterization call to limit VRAM usage [S1, unofficial].
- `texture_size`: Resolution of the 2D buffer. Not specified in sources, but must be large enough to resolve the mesh details.

**Edge Cases and Pitfalls:**
- **Face ID Overwrite:** The face ID is stored in the alpha channel by adding the loop index $i$. This overwrites the original alpha value (which might represent depth or coverage weight). The mask must be recomputed or stored separately before merging if the alpha channel is used for other purposes in the rasterizer [S1, unofficial].
- **Precision:** Using float32 for face IDs may lose precision for meshes with more than $2^{24}$ faces. The source uses integer addition to the alpha channel, which is stored as float32. For very large meshes, consider using a separate buffer for face IDs or a different encoding [S1, unofficial].
- **Boundary Conditions:** The UV coordinates are mapped to $[-1, 1]$. Ensure that the mesh's UV layout fits within this range and does not have extreme stretching that could cause rasterization artifacts.

## Subtopics

### Face Token Layout
A `uint64_t` face token encodes a unit face with specific bit fields for compact storage and fast lookup [S2, unofficial].
- **Bits 0-5:** u coordinate (6 bits).
- **Bits 6-11:** v coordinate (6 bits).
- **Bits 12-17:** depth (6 bits).
- **Bits 18-20:** axis (3 bits).
- **Bit 21:** positive direction flag (1 bit).
- **Bits 22-31:** material (10 bits).
This layout allows for efficient packing of face attributes into a single 64-bit integer, facilitating fast sorting and deduplication in greedy meshing algorithms [S2, unofficial].

### Face Specification for Padded Coordinates
A struct encoding the orientation of a face within a chunk and its matching padded boundary index [S3, unofficial].
- **Structure:** `FaceSpec` contains `fixed_axis: usize`, `fixed_value: u8`, `iter_axes: [usize; 2]`, and `padded_boundary_index: u32`.
- **Formula:** Padded coordinates for a face point $(u, v)$ are calculated as:
  $$
  \text{coords}[\text{iter\_axes}[0]] = u + 1
  $$
  $$
  \text{coords}[\text{iter\_axes}[1]] = v + 1
  $$
  $$
  \text{coords}[\text{fixed\_axis}] = \text{padded\_boundary\_index}
  $$
This specification is used to map 2D rasterized coordinates to 3D voxel positions within a chunked grid, handling boundary conditions and memory alignment [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_f \cdot R)$, where $N_f$ is the number of faces and $R$ is the rasterization cost per face (dependent on texture size and rasterizer efficiency). The chunking does not change the asymptotic complexity but improves cache locality and VRAM usage.
- **Memory Complexity:** $O(\text{texture\_size}^2)$ for the rasterization buffer, plus $O(N_v)$ for vertex data. The chunking allows processing meshes larger than VRAM allows for a single rasterization pass.
- **Performance:** The method leverages efficient 2D rasterization libraries (e.g., DiffRast) which are highly optimized on GPUs. The chunk size of 100,000 faces is a practical trade-off between memory usage and kernel launch overhead [S1, unofficial].
- **Strengths:** Memory efficient for large meshes; leverages existing 2D rasterization infrastructure; simple to implement using differentiable rendering libraries.
- **Limitations:** Requires UV coordinates for the mesh; rasterization artifacts (e.g., z-fighting, coverage errors) can affect voxel quality; not suitable for meshes without valid UVs.

## Background
This method is derived from the implementation in the `o-voxel-gpu` repository, which uses differentiable rasterization for voxelization [S1, unofficial]. It represents a practical approach to voxelizing large-scale meshes by breaking down the problem into manageable chunks, avoiding the need for large 3D buffers during the rasterization phase.

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S3] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
