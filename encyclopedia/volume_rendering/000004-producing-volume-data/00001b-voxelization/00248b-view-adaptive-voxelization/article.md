# View-Adaptive Voxelization

Also called: Tiled View-Adaptive Voxelization, on-the-fly voxelization, tiled-based voxelization

## Overview
View-Adaptive Voxelization is a GPU-friendly, real-time technique for rendering homogeneous media defined by particles (e.g., SPH fluids) by computing view-adaptive binary voxelizations with on-the-fly compression [S1]. It solves the problem of rendering high-resolution volumetric data that exceeds available GPU memory by splitting the screen into tiles, voxelizing each tile into a perspective grid, and immediately compressing the result into surface spans [S1]. Unlike [FAW10] or [RTW13], it requires no preprocessing such as scalar field conversion or mesh generation, allowing direct rendering of unsorted particle data [S1]. It sets itself apart from siblings like [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by prioritizing memory efficiency through immediate compression and view-adaptivity rather than storing a full static grid, and from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) by explicitly handling depth layers for translucency and refraction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Unsorted list of particles (position, radius). Optionally, simulation grid information to identify fully populated cells [S1].
*   **Output:** Compressed surface span data (depths and normals) per pixel, suitable for translucency or refraction rendering [S1].

**Data Structures:**
1.  **Bit-packed Perspective Voxel Grid:**
    *   Layout: 3D array of bits.
    *   Dimensions: $W_{tile} \times H_{tile} \times L$, where $L=16384$ layers.
    *   Storage: 1 bit per voxel (0=empty, 1=occupied) [S1].
    *   Typical Tile Size: $1024 \times 512$ pixels, consuming ~1024 MB [S1].
2.  **Surface Span Storage:**
    *   Layout: Array of spans per pixel at full screen resolution.
    *   Content: Entry and exit depths, and surface normals (replacing surface links) [S1].
    *   Capacity: Up to 7 spans per pixel [S1].
    *   Size: ~126 MB for depths/normals, totaling 252 MB at 1920×1080 [S1].

**Parameters:**
*   **Angle Resolution:** $14^\circ$. This determines the number of depth layers $L$. $L \approx 16384$ covers the near-to-far plane range [S1].
*   **Max Iterations for Filtering:** 20 [S1].
*   **Max Spans per Pixel:** 7 (stored explicitly) [S1].

**Steps:**

1.  **Tile Partitioning:** Split the screen into tiles of size $W_{tile} \times H_{tile}$ such that the voxel grid for each tile fits in GPU memory [S1].
2.  **Voxel Grid Initialization:** For each tile, allocate a bit-packed 3D grid of size $W_{tile} \times H_{tile} \times 16384$ [S1].
3.  **Splatting:**
    *   For each particle, project its sphere onto the screen tile.
    *   Identify all pixels (columns) overlapped by the particle.
    *   For each overlapped pixel, mark the voxels in the column corresponding to the particle's depth range as occupied (1) [S1].
    *   *Optimization:* If simulation grid data is available, skip particles in fully populated inner cells [S1].
4.  **Surface Extraction (Compression):**
    *   Iterate over all columns in the voxel grid in parallel.
    *   For each column, scan the bits to find entry and exit depths of occupied regions (spans) [S1].
    *   Store these spans (depths and normals) in the Surface Span Storage [S1].
    *   Handle overflow: If the number of layers exceeds the allocated span capacity (e.g., >7 spans), track the actual number of layers and the empty space in the last extended span to preserve volumetric absorption for translucency rendering [S1].
5.  **Discard and Repeat:** Discard the voxel subgrid for the current tile. Repeat steps 2-4 for all tiles [S1].
6.  **Rendering:**
    *   **Translucency:** Iterate sequentially over all surfaces extracted from each voxel column [S1].
    *   **Refraction:** Cast perturbed refraction rays that traverse multiple voxel columns [S1].
    *   **Acceleration:** Optionally build a hierarchical span structure (mipmapping-like) where boundary surfaces of four neighboring columns are conservatively bounded in a coarser grid column [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_adaptive_voxelization(particles, screen_width, screen_height, tile_width=1024, tile_height=512, num_layers=16384):
    """
    Performs view-adaptive voxelization on a tiled basis.
    """
    # Calculate number of tiles
    num_tiles_x = (screen_width + tile_width - 1) // tile_width
    num_tiles_y = (screen_height + tile_height - 1) // tile_height
    
    # Initialize global surface storage (e.g., 7 spans per pixel)
    # Span structure: (entry_depth, exit_depth, normal_x, normal_y, normal_z)
    surface_spans = [[[] for _ in range(screen_width)] for _ in range(screen_height)]
    
    for ty in range(num_tiles_y):
        for tx in range(num_tiles_x):
            # 1. Initialize tile voxel grid (bit-packed)
            # Dimensions: tile_width x tile_height x num_layers
            # Using a flat list for bit packing: 1 bit per voxel
            tile_grid_size = tile_width * tile_height * num_layers
            voxel_grid = [0] * (tile_grid_size // 8 + 1) # Byte array for bits
            
            # 2. Splatting
            for particle in particles:
                # Project particle sphere to screen tile coordinates
                # Determine bounding box in tile pixel space and depth range
                tile_pixels = get_overlapping_pixels(particle, tx * tile_width, ty * tile_height, tile_width, tile_height)
                
                for (px, py) in tile_pixels:
                    # Determine depth range [z_min, z_max] for this particle in this column
                    z_min, z_max = get_particle_depth_range(particle, px, py)
                    
                    # Mark voxels in the column
                    for z in range(z_min, z_max + 1):
                        set_bit(voxel_grid, px, py, z, 1)
            
            # 3. Surface Extraction
            for px in range(tile_width):
                for py in range(tile_height):
                    # Extract spans from the column
                    spans = extract_spans_from_column(voxel_grid, px, py, num_layers)
                    
                    # Map to global screen coordinates
                    global_px = tx * tile_width + px
                    global_py = ty * tile_height + py
                    
                    # Store spans, handling overflow if necessary
                    for span in spans:
                        add_span(surface_spans[global_py][global_px], span)
            
            # 4. Discard voxel grid for next tile
            voxel_grid = None

    return surface_spans

def extract_spans_from_column(voxel_grid, px, py, num_layers):
    """
    Scans a voxel column to find entry/exit depths.
    Returns list of (entry, exit, normal) tuples.
    """
    spans = []
    current_entry = None
    
    for z in range(num_layers):
        is_occupied = get_bit(voxel_grid, px, py, z)
        
        if is_occupied and current_entry is None:
            current_entry = z
        elif not is_occupied and current_entry is not None:
            # Exit found
            exit_z = z - 1
            normal = compute_normal(current_entry, exit_z) # Placeholder for normal computation
            spans.append((current_entry, exit_z, normal))
            current_entry = None
            
    # Handle case where column ends while occupied
    if current_entry is not None:
        exit_z = num_layers - 1
        normal = compute_normal(current_entry, exit_z)
        spans.append((current_entry, exit_z, normal))
        
    return spans

def add_span(pixel_spans, new_span):
    """
    Adds a span to the pixel's list, handling the 7-span limit.
    If limit exceeded, merges or extends the last span to preserve absorption.
    """
    if len(pixel_spans) < 7:
        pixel_spans.append(new_span)
    else:
        # Overflow handling: extend the last span to cover remaining depth
        # and track empty space if needed for translucency
        last_span = pixel_spans[-1]
        # Extend exit depth to include new span's exit
        last_span.exit_depth = max(last_span.exit_depth, new_span.exit_depth)
        # Note: Normal might need averaging or conservative update [G]
```

## Subtopics

### Hierarchical Span Acceleration
A method to accelerate ray casting by building a mipmapping-like hierarchy where boundary surfaces of four neighboring columns are conservatively bounded in one column of a coarser grid [S1].

### Translucency-Only Rendering
A rendering mode that iterates sequentially over all extracted surfaces from a single voxel column, avoiding the cost of refraction ray casting [S1].

### Refraction Rendering
A rendering mode that casts perturbed refraction rays, potentially traversing multiple voxel columns, increasing computational cost but enabling complex light effects [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by splatting and surface extraction (RLE compression). Performance scales with particle count and tile size [S1].
*   **Memory Complexity:** $O(W_{tile} \times H_{tile} \times L)$ for the voxel grid, where $L=16384$. For a $1024 \times 512$ tile, this is ~1024 MB. Surface storage is $O(W \times H \times S)$, where $S=7$ spans [S1].
*   **Measured Performance:** On an NVIDIA GTX Titan, voxelization and translucency rendering of 7.1 million particles took 187 ms [S1]. This is ~10x faster than [FAW10] (2514 ms for 8.7M particles) and ~2x faster than [RTW13] [S1].
*   **Strengths:** High resolution (up to 3840×2160×16000), no preprocessing, view-adaptive memory usage, supports complex illumination (refraction, reflection) [S1].
*   **Limitations:** Limited to homogeneous media (only entry/exit points detected, no scalar attributes per layer) [S1]. Refraction rendering is expensive. View-adaptivity makes non-view-dependent tasks (e.g., shadow mapping) require a second voxelization [S1].
*   **When to Use:** Real-time rendering of particle-based homogeneous media (SPH fluids) where high resolution and visual quality are prioritized over heterogeneous attribute storage [S1].
*   **When Not to Use:** Heterogeneous media requiring per-layer scalar attributes, or scenes requiring frequent non-view-dependent volumetric queries [S1].

## Background
View-Adaptive Voxelization was introduced in "Memory-Efficient On-The-Fly Voxelization of Particle Data" (2015) [S1]. It addresses the limitations of previous methods like [FAW10] (which requires preprocessing) and [RTW13] (which requires extensive preprocessing and targets out-of-core rendering) by providing a real-time, on-the-fly solution [S1]. It is particularly suited for SPH fluid simulation visualization [S1].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
