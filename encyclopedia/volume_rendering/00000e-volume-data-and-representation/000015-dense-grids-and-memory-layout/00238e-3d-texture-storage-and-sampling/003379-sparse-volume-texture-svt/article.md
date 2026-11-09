# Sparse Volume Texture (SVT)
Also called: SVT in UE5, Sparse Volume Texture (SVT) in UE5

## Overview
Sparse Volume Texture (SVT) is a technique that trades rendering speed for data compression by chunking voxel data into 3D tiles and storing only non-empty tiles in a physical texture, using a second volume texture as a page table for lookup [S1, unofficial]. It solves the problem of rendering large volume datasets that exceed the maximum dimension limits of standard 3D textures (e.g., 2048 voxels per dimension) while managing memory footprint [S1, unofficial]. Unlike dense grids, SVT circumvents API limits by mapping a large logical coordinate space onto a smaller physical texture containing only active tiles, though it incurs overhead from padding and mipmap hierarchies [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A large voxel dataset (logical coordinates) [S1, unofficial].
- Tile size: $16^3$ voxels [S1, unofficial].
- Maximum texture dimension: 2048 [S1, unofficial].
- Padding: Required for trilinear interpolation at tile borders [S1, unofficial].

**Outputs:**
- A physical tile data texture containing packed, padded $16^3$ tiles [S1, unofficial].
- A page table texture mapping logical tile coordinates to physical offsets [S1, unofficial].
- A mipmap hierarchy stored within the same texture as the base layer [S1, unofficial].

### Data Structures
1.  **Physical Tile Data Texture:**
    -   Format: 3D Texture [S1, unofficial].
    -   Dimensions: Up to $2048^3$ texels [S1, unofficial].
    -   Content: Densely packed $16^3$ tiles. Each tile is padded to allow correct trilinear interpolation at borders [S1, unofficial].
    -   Mipmaps: The mipmap hierarchy is stored in the same texture object as the base layer [S1, unofficial].
2.  **Page Table Texture:**
    -   Format: 3D Texture [S2].
    -   Content: Stores offsets or indices to locate non-empty tiles within the Physical Tile Data Texture [S1, unofficial].
3.  **CPU Upload Buffer:**
    -   Format: ByteAddressBuffer (GPU access) / uint32 offsets (CPU side) [S1, unofficial].
    -   Limit: 32-bit unsigned integer range ($2^{32}$ bytes = 4 GB) [S1, unofficial].

### Steps
1.  **Chunking:** Divide the logical voxel volume into $16^3$ tiles [S1, unofficial].
2.  **Filtering:** Identify non-empty tiles. Empty tiles are not stored in the physical texture [S1, unofficial].
3.  **Padding:** Pad each $16^3$ tile to enable correct trilinear interpolation at tile borders [S1, unofficial].
4.  **Packing:** Pack the padded, non-empty tiles densely into the Physical Tile Data Texture [S1, unofficial].
5.  **Mipmap Generation:** Generate a mipmap hierarchy for the packed texture. The mipmaps are stored in the same texture object [S1, unofficial].
6.  **Page Table Construction:** Construct a page table (second volume texture) that maps logical tile coordinates to the physical offsets of the packed tiles in the Physical Tile Data Texture [S1, unofficial].
7.  **Upload:** Upload CPU-side data to the GPU. The upload buffer uses `uint32` offsets [S1, unofficial]. The implementation processes chunks in batches, with a limit of 227 elements uploaded at a single time, implying a sliding window approach [S1, unofficial].

### Formulas
**Usable Payload:**
The usable payload in the texture, accounting for padding, is:
$$ \text{Payload} = \frac{2048^3 \times 16^3}{(1 + 16 + 1)^3} \approx 5.6 \text{ gigavoxels} $$
[S1, unofficial]

**Net Payload:**
The net payload after mipmap overhead (factor of $1.143$) is:
$$ \text{Net Payload} = \frac{2048^3 \times 16^3}{(1 + 16 + 1)^3 / 1.143} \approx 4.9 \text{ G} $$
[S1, unofficial]

**Memory Footprint:**
The memory footprint increases by a factor of 4 compared to a scalar density value due to the overhead of the page table and tile structure [S1, unofficial].

### Parameters
-   **Tile Size:** $16^3$ voxels [S1, unofficial].
-   **Max Texture Dimension:** 2048 voxels per dimension [S1, unofficial].
-   **Lighting Cache Downsample Factor:** 4 (`r.HeterogeneousVolumes.LightingCache.DownsampleFactor`) [S1, unofficial].
-   **Max Step Count:** 1024 steps per ray (`r.HeterogeneousVolumes.MaxStepCount`) [S1, unofficial].
-   **Batch Upload Limit:** 227 elements per upload call [S1, unofficial].

### Edge Cases and Pitfalls
-   **Signed Int32 Overflow:** The engine crashes at approximately 2 gigavoxels of non-empty compressed voxel data due to signed int32 arithmetic overflow ($2^{31} = 2$ GB) in the source code [S1, unofficial]. This is significantly lower than the theoretical limit of 4.9 gigavoxels [S1, unofficial].
-   **Unsigned Int32 Overflow:** A potential unsigned uint32 overflow exists in the RAM-VRAM upload code at $2^{32} = 4$ GB, limiting the maximum texture size to approximately 8 GB for rendering [S1, unofficial]. The code explicitly catches this with the error "SVT streaming data overflowed the uint32 range!" [S1, unofficial].
-   **API Limit:** Expanding the upload buffer beyond 4 GB only benefits high-occupancy tiles; sparse datasets hit the 2048-dimension API limit due to empty voxels in padded tiles [S1, unofficial].
-   **Dense Datasets:** For dense datasets (e.g., Kolumbo, ~70% average occupancy), SVT provides no memory or performance gain over dense textures, but successfully circumvents the 2048 voxels/dimension API limit [S1, unofficial].
-   **Out-of-Core Requirement:** The in-core SVT approach is insufficient for true large-scale rendering; a true out-of-core implementation is required to overcome the 4GB/2GB limits [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def create_padded_tile(tile_size, voxel_data):
    """Creates a padded tile buffer for trilinear interpolation."""
    padded_size = tile_size + 2
    buffer = [0.0] * (padded_size ** 3)
    for lx, ly, lz, val in voxel_data:
        idx = (lz + 1) * padded_size * padded_size + (ly + 1) * padded_size + (lx + 1)
        buffer[idx] = val
    return buffer

def create_byte_address_buffer(tiles):
    """Concatenates tile buffers into a single GPU-accessible buffer."""
    buffer = []
    for tile in tiles:
        buffer.extend(tile)
    return buffer

def create_texture(page_table):
    """Creates a 3D texture from the page table mapping."""
    return page_table

def upload_svt_data(logical_voxels, tile_size=16):
    """
    Uploads sparse voxel data to GPU.
    logical_voxels: List of (x, y, z, value) tuples.
    tile_size: Size of each tile (16).
    """
    tiles = {}
    for x, y, z, value in logical_voxels:
        tx, ty, tz = x // tile_size, y // tile_size, z // tile_size
        if (tx, ty, tz) not in tiles:
            tiles[(tx, ty, tz)] = []
        tiles[(tx, ty, tz)].append((x % tile_size, y % tile_size, z % tile_size, value))

    physical_tiles = []
    page_table = {}
    current_offset = 0

    sorted_tile_keys = sorted(tiles.keys())

    for tx, ty, tz in sorted_tile_keys:
        voxel_data = tiles[(tx, ty, tz)]
        if not voxel_data:
            continue

        padded_tile = create_padded_tile(tile_size, voxel_data)

        if current_offset + len(padded_tile) > 2**32:
            raise OverflowError("SVT streaming data overflowed the uint32 range!")

        physical_tiles.append(padded_tile)
        page_table[(tx, ty, tz)] = current_offset
        current_offset += len(padded_tile)

    upload_buffer = create_byte_address_buffer(physical_tiles)
    page_table_texture = create_texture(page_table)

    return upload_buffer, page_table_texture

def render_svt_volume(ray_origin, ray_dir, step_size, max_steps, page_table, physical_texture):
    """
    Renders volume using SVT.
    ray_origin: (x, y, z) tuple.
    ray_dir: (dx, dy, dz) tuple.
    step_size: Distance per step.
    max_steps: Maximum ray marching steps.
    page_table: Dict mapping (tx, ty, tz) to byte offset.
    physical_texture: Concatenated tile buffers.
    """
    t = 0.0
    accumulated_color = [0.0, 0.0, 0.0, 0.0]
    accumulated_density = 0.0
    alpha = 1.0

    for _ in range(max_steps):
        pos = [ray_origin[i] + ray_dir[i] * t for i in range(3)]
        tile_coords = [int(pos[i] // 16) for i in range(3)]
        local_coords = [(pos[i] - tile_coords[i] * 16) / 16.0 for i in range(3)]

        # Clamp to [0, 1] to correctly sample padded borders
        local_coords = [max(0.0, min(1.0, c)) for c in local_coords]

        offset = page_table.get(tuple(tile_coords), -1)
        if offset == -1:
            t += step_size
            continue

        # Fetch tile data and sample
        tile_size = 16
        padded_size = tile_size + 2
        tile_buffer = physical_texture[offset:offset + padded_size**3]

        # Trilinear interpolation simulation
        u, v, w = local_coords
        x0, y0, z0 = int(u * tile_size), int(v * tile_size), int(w * tile_size)
        x1, y1, z1 = min(x0 + 1, tile_size), min(y0 + 1, tile_size), min(z0 + 1, tile_size)

        def sample_texel(lx, ly, lz):
            idx = (lz + 1) * padded_size * padded_size + (ly + 1) * padded_size + (lx + 1)
            return tile_buffer[idx]

        c000 = sample_texel(x0, y0, z0)
        c100 = sample_texel(x1, y0, z0)
        c010 = sample_texel(x0, y1, z0)
        c110 = sample_texel(x1, y1, z0)
        c001 = sample_texel(x0, y0, z1)
        c101 = sample_texel(x1, y0, z1)
        c011 = sample_texel(x0, y1, z1)
        c111 = sample_texel(x1, y1, z1)

        # Interpolate along x
        c00 = [c000[i] + u * (c100[i] - c000[i]) for i in range(4)]
        c01 = [c010[i] + u * (c110[i] - c010[i]) for i in range(4)]
        c10 = [c001[i] + u * (c101[i] - c001[i]) for i in range(4)]
        c11 = [c101[i] + u * (c111[i] - c101[i]) for i in range(4)]

        # Interpolate along y
        c0 = [c00[i] + v * (c01[i] - c00[i]) for i in range(4)]
        c1 = [c10[i] + v * (c11[i] - c10[i]) for i in range(4)]

        # Interpolate along z
        sample = [c0[i] + w * (c1[i] - c0[i]) for i in range(4)]

        density = sample[0]
        color = sample[1:4]
        alpha_val = sample[3]

        transmittance = math.exp(-accumulated_density * step_size)
        contribution = [color[i] * (1.0 - transmittance) for i in range(3)]
        accumulated_color = [accumulated_color[i] + contribution[i] * alpha for i in range(3)]
        accumulated_color[3] = alpha * transmittance
        alpha *= transmittance
        accumulated_density += density * step_size

        if alpha < 0.001:
            break
        t += step_size

    return accumulated_color
```

## Subtopics

### SVT Streaming Optimization
Techniques to optimize real-time playback of animated SVTs by managing resolution, sparseness, and streaming prefetch to prevent fidelity drops [S2]. Use a smaller pixel format when importing a VDB (e.g., avoid 32-bit) to improve streaming performance, lower the animation frame rate to reduce streaming load, or use a lower resolution mip level by setting the MipLevel on the Animated SVT Controller or Heterogeneous Volume component settings [S2]. The console variable `r.SparseVolumeTexture.Streaming.NumPrefetchFrames` controls the number of frames to prefetch when a frame is requested (default is 3) [S2]. Setting `NumPrefetchFrames` too high can choke the streaming system with requests, while setting it too low might cause frames to not be fully streamed in by the time they are needed for rendering [S2].

### Heterogeneous Volumes
A rendering approach in Unreal Engine 5 that uses SVT for large datasets, supporting interactive exploration and lighting [S1, unofficial].

### Sparse Textures (AMD)
An earlier technique using partially-resident textures to mark empty regions as non-resident, falling short for 3D textures due to minimum brick sizes [S3, unofficial].

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** The memory footprint increases by a factor of 4 compared to a scalar density value [S1, unofficial].
-   **Measured Performance:** Renders a 5.9 gigavoxel sparse dataset (8 bit/voxel) at ~10 fps on RTX 5000 and ~20 fps on RTX 3500 Ada Laptop GPU [S1, unofficial].
-   **Scaling:** Extending the upload buffer beyond 4 GB provides limited benefit (factor of ~1.31) for datasets with high per-tile occupancy (e.g., Kolumbo dataset, ~70% average occupancy) or high bit resolution, but little to no benefit for less compact or lower bit resolution datasets [S1, unofficial].
-   **Strengths:** Circumvents the 2048 voxels/dimension API limit; reduces displayed volume size by a factor of $4^3 = 64$ compared to the original dataset [S1, unofficial].
-   **Limitations:** Crashes at ~2 gigavoxels due to signed int32 overflow; CPU upload buffer limited to uint32 range (4 GB); ineffective for dense datasets where memory/performance gains are negated [S1, unofficial].
-   **When to Use:** For large, sparse volume datasets that exceed the 2048 dimension limit in Unreal Engine 5 [S1, unofficial].
-   **When Not to Use:** For dense datasets (e.g., Kolumbo) where SVT offers no memory or performance gain over dense textures [S1, unofficial].

## Background
Sparse Volume Texture (SVT) is an in-core implementation used for rendering large volume datasets in Unreal Engine 5, compressing voxel data into tiles to reduce memory footprint [S1, unofficial]. It was evaluated with the Kolumbo and Manganese nodule scans [S1, unofficial]. The technique represents a trade-off between rendering speed and data compression using a page table approach [S1, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://arxiv.org/abs/2504.07485, fetched 2026-10-27 (unofficial)
[S2] Sparse Volume Textures in Unreal Engine | Unreal Engine 5.8 ... - Epic Dev, https://dev.epicgames.com/documentation/en-us/unreal-engine/sparse-volume-textures-in-unreal-engine, fetched 2026-10-28
[S3] http://realtimevoxels.blogspot.fr/2012/09/introduction-to-gigavoxels-and-sparse.html, http://realtimevoxels.blogspot.fr/2012/09/introduction-to-gigavoxels-and-sparse.html, fetched 2026-10-28 (unofficial)
