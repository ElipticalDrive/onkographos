# View-Adaptive Voxelization

Also called: Tiled View-Adaptive Voxelization, on-the-fly voxelization, tiled-based voxelization

## Overview
View-Adaptive Voxelization is a GPU-friendly, real-time technique for rendering homogeneous media defined by particles (e.g., SPH fluids) by computing view-adaptive binary voxelizations with on-the-fly compression [S1]. It solves the problem of high-resolution volumetric data exceeding available GPU memory by splitting the screen into tiles, voxelizing each tile at full resolution, and immediately compressing the result into a span-based surface representation before discarding the voxel grid [S1]. Unlike [FAW10] or [RTW13], it requires no preprocessing such as scalar field conversion or mesh generation, allowing direct rendering of unsorted particle data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Unsorted list of particles (position, radius) [S1]. Optionally, simulation grid metadata to identify fully populated cells [S1].
*   **Output:** A compressed surface representation consisting of entry/exit depths and normals for each pixel, suitable for translucency or refraction rendering [S1].

**Data Structures:**
1.  **Bit-packed Perspective Voxel Grid (Per Tile):**
    *   Dimensions: $W_{tile} \times H_{tile} \times L$, where $L=16384$ layers [S1].
    *   Storage: 1 bit per voxel (0=empty, 1=occupied) [S1].
    *   Memory: ~1024 MB for a $1024 \times 512$ tile [S1].
2.  **Surface Span Storage (Global):**
    *   Dimensions: $W_{screen} \times H_{screen} \times N_{spans}$, where $N_{spans} \le 7$ [S1].
    *   Fields per span: Depth ($f32$), Normal ($f32 \times 3$) [S1].
    *   Memory: ~252 MB for $1920 \times 1080$ [S1].
    *   Note: Surface links are replaced by normals [S1].

**Parameters:**
*   **Angle Resolution:** $14^\circ$ [S1].
*   **Depth Layers:** $L = 16384$ [S1].
*   **Max Iterations (Filtering):** 20 [S1].
*   **Max Spans per Pixel:** 7 (with overflow handling) [S1].

**Algorithm Steps:**

1.  **Tile Partitioning:** Split the screen into tiles of size $1024 \times 512$ (optimal for GTX 970/Titan) [S1].
2.  **Voxelization (Splatting):** For each tile:
    *   Initialize the $W_{tile} \times H_{tile} \times 16384$ bit grid to 0 [S1].
    *   For each particle $p_i$ with position $\mathbf{p}_i$ and radius $r_i$:
        *   Project $p_i$ onto the tile's pixel grid [S1].
        *   Determine the range of depth layers $[z_{min}, z_{max}]$ intersected by the particle sphere along the view ray for each affected pixel [S1].
        *   Set bits in the grid for all layers in $[z_{min}, z_{max}]$ to 1 [S1].
    *   *Optimization:* If simulation grid data is available, skip particles in fully populated inner cells [S1].
3.  **Compression (Surface Extraction):**
    *   For each pixel $(u, v)$ in the tile, iterate through the 16384 layers in parallel [S1].
    *   Identify transitions from 0 to 1 (entry) and 1 to 0 (exit) [S1].
    *   Store up to 7 entry/exit span pairs (depth + normal) in the global Surface Span Storage [S1].
    *   *Overflow Handling:* If layers > 7 spans fit:
        *   Track the actual number of layers [S1].
        *   Extend the last span to include the amount of empty space between the last encountered layer and the end of the grid [S1].
        *   This allows correct volumetric absorption for translucency-only rendering [S1].
    *   Discard the tile's voxel grid [S1].
4.  **Normal Generation:**
    *   Compute normals for each span by connecting entry and exit surface points [S1].
5.  **Filtering:**
    *   Apply iterative filtering (up to 20 iterations) based on viewer distance [S1].
6.  **Rendering:**
    *   **Translucency:** Iterate sequentially over all spans in a voxel column [S1].
    *   **Refraction:** Cast perturbed refraction rays traversing multiple columns [S1].
    *   **Acceleration:** Use a hierarchical span representation (mipmapping-like) where boundary surfaces of four neighboring columns are conservatively bounded in one column of a coarser grid [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_adaptive_voxelization(particles, screen_width, screen_height, tile_w=1024, tile_h=512, layers=16384):
    # Global storage for compressed surfaces
    # Structure: list of lists of spans. spans[u][v] = [(depth, normal), ...]
    surface_spans = [[[] for _ in range(screen_height)] for _ in range(screen_width)]
    
    # Tile dimensions
    num_tiles_x = (screen_width + tile_w - 1) // tile_w
    num_tiles_y = (screen_height + tile_h - 1) // tile_h
    
    for ty in range(num_tiles_y):
        for tx in range(num_tiles_x):
            # 1. Initialize Tile Voxel Grid (Bit-packed)
            # Dimensions: tile_w x tile_h x layers
            # Using a flat array for bit packing: index = (z * tile_h + y) * tile_w + x
            tile_grid_size = tile_w * tile_h * layers
            tile_grid = [0] * tile_grid_size
            
            # 2. Splatting
            for particle in particles:
                # Project particle to tile coordinates
                # Determine pixel range (u_start, u_end, v_start, v_end) within tile
                # Determine depth range (z_min, z_max) for each pixel in range
                # This step involves sphere-ray intersection logic
                for u in range(u_start, u_end):
                    for v in range(v_start, v_end):
                        # Calculate z_min, z_max for this pixel and particle
                        for z in range(z_min, z_max):
                            # Set bit in tile_grid
                            idx = (z * tile_h + v) * tile_w + u
                            tile_grid[idx] = 1
            
            # 3. Compression / Surface Extraction
            for u in range(tile_w):
                for v in range(tile_h):
                    spans = []
                    current_span_start = None
                    layer_count = 0
                    
                    # Iterate through layers
                    for z in range(layers):
                        idx = (z * tile_h + v) * tile_w + u
                        is_occupied = tile_grid[idx]
                        
                        if is_occupied and current_span_start is None:
                            # Entry
                            current_span_start = z
                            layer_count = 1
                        elif is_occupied and current_span_start is not None:
                            layer_count += 1
                        elif not is_occupied and current_span_start is not None:
                            # Exit
                            exit_z = z
                            # Compute normal from entry/exit points
                            normal = compute_normal(current_span_start, exit_z)
                            spans.append((current_span_start, normal))
                            current_span_start = None
                            layer_count = 0
                    
                    # Handle case where last layer is occupied
                    if current_span_start is not None:
                        exit_z = layers
                        normal = compute_normal(current_span_start, exit_z)
                        spans.append((current_span_start, normal))
                        # Overflow handling: if len(spans) > 7, merge/extend last span
                        if len(spans) > 7:
                            # Keep first 7, extend the 7th to cover remaining empty space
                            # The source notes: "keep track of the actual number of layers 
                            # and the amount of empty space inside the extended last span"
                            # Implementation detail: The 7th span's depth is extended, 
                            # and metadata tracks the gap.
                            pass 
                    
                    # Store in global structure
                    global_u = tx * tile_w + u
                    global_v = ty * tile_h + v
                    surface_spans[global_u][global_v] = spans

    # 4. Filtering and Normal Generation (Post-processing)
    # Apply iterative filtering based on viewer distance
    # Generate normals if not done during extraction
    return surface_spans

def compute_normal(z_entry, z_exit):
    # Placeholder for normal calculation based on ray direction and surface geometry
    # In the source, normals are derived from connecting entry/exit points
    return [0, 0, 1] # [G] Simplified normal
```

## Operating characteristics
*   **Time Complexity:** Dominated by splatting and RLE compression [S1]. Performance scales with particle count and tile size [S1].
*   **Memory Complexity:** $O(W_{tile} \times H_{tile} \times L)$ for the temporary voxel grid + $O(W_{screen} \times H_{screen} \times N_{spans})$ for output [S1].
*   **Measured Performance:** 187 ms for 7.1M particles on a GTX Titan for translucency rendering [S1]. 10x faster than [FAW10] and 2x faster than [RTW13] [S1].
*   **Strengths:** No preprocessing; high resolution (up to 3840×2160×16000); view-adaptive memory usage [S1].
*   **Limitations:** Homogeneous media only (no scalar attributes); refraction is expensive; view-dependent (requires recomputation for different viewpoints) [S1].

## Background
Originating from the paper "Memory-Efficient On-The-Fly Voxelization of Particle Data" [S1], this method addresses the bottleneck of rendering SPH simulations. Previous methods like [FAW10] required preprocessing, and [RTW13] required 35 hours of preprocessing for out-of-core rendering [S1]. This approach enables interactive visualization by leveraging GPU parallelism for on-the-fly compression [S1].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
