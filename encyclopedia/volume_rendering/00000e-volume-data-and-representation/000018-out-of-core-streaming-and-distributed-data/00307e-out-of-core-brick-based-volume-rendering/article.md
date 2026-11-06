# Out-of-core brick-based volume rendering

Also called: Brick pyramid rendering, Volume virtualization

## Overview
Out-of-core brick-based volume rendering is a technique that streams multi-gigabyte volumetric datasets over HTTP and renders them interactively with a fixed VRAM budget by decoupling logical 3D grid coordinates from physical GPU memory using a brick pyramid and indirection texture [S1, unofficial]. It solves the problem of rendering datasets too large to fit in memory or download monolithically by streaming only the visible bricks relevant to the current camera view [S1, unofficial]. Unlike monolithic download approaches, it avoids downloading the entire dataset before rendering a single pixel, relying instead on HTTP range reads to fetch data on demand [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A source volumetric dataset (e.g., raw 16-bit data, OME-Zarr, or custom binary) and a camera view (position, orientation, field of view).
*   **Output:** A rendered image frame.

**Data Structures:**
1.  **Brick Pyramid:** A multi-resolution hierarchy where the finest level contains the full-resolution data and coarser levels are downsampled versions. The volume is split into fixed-size chunks called bricks.
    *   **Logical Brick Size:** $64^3$ voxels.
    *   **Physical Brick Size:** $66^3$ voxels (to facilitate interpolation during sampling) [S1, unofficial].
    *   **Storage:** Compressed bricks stored server-side in structured binary blobs [S1, unofficial].
2.  **GPU Page Cache (Atlas):** A fixed-size cache of slots holding decompressed bricks.
    *   **Default Size:** 512 slots [S1, unofficial].
    *   **Scalable Size:** Up to $768^3$ or $1024^3$ slots for larger volumes or high-DPI screens [S1, unofficial].
    *   **VRAM Footprint:** Approximately 300 MB at 512 slots for 16-bit data, remaining fixed regardless of total dataset size [S1, unofficial].
3.  **Indirection Texture:** A compact 2D/3D texture storing cache slot indices. Each texel corresponds to a logical brick coordinate in the pyramid and points to its current physical location in the GPU page cache [S1, unofficial].

**Algorithm Steps:**
1.  **Pre-processing:** Split the source dataset into fixed-size chunks ($64^3$ logical, $66^3$ physical). Generate downsampled LOD levels for the brick pyramid. Compress the bricks and store them server-side [S1, unofficial].
2.  **View Frustum Culling:** Identify bricks relevant to the current camera view. Determine the appropriate LOD level for each brick based on distance from the camera (coarse LOD for far bricks, fine LOD for close bricks) [S1, unofficial].
3.  **Fetching:** Fetch the identified bricks from the server using HTTP range-reads [S1, unofficial].
4.  **Decompression:** Decompress the fetched bricks asynchronously in dedicated worker threads [S1, unofficial].
5.  **Upload and Cache Update:** Upload the decompressed data to the GPU. Update the GPU page cache:
    *   If the target slot is free, place the brick there.
    *   If the target slot is occupied, evict the existing brick using a Least Recently Used (LRU) strategy [S1, unofficial].
6.  **Indirection Update:** Update the indirection texture to map the logical brick coordinates of the newly cached bricks to their physical cache slot indices [S1, unofficial].
7.  **Ray Marching:** During compute shader ray marching, for each sample along a ray:
    *   Resolve the indirection texture using normalized box coordinates to locate the correct brick in the cache [S1, unofficial].
    *   Perform the density lookup using local brick coordinates within the located brick [S1, unofficial].
8.  **Compositing:** Composite the density samples along the ray to produce the final pixel color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BRICK_LOGICAL_SIZE = 64
BRICK_PHYSICAL_SIZE = 66
CACHE_SIZE = 512

# Data Structures
# IndirectionTexture: 3D texture mapping (brick_x, brick_y, brick_z) -> cache_slot_index
# Cache: Array of size CACHE_SIZE, each element holds a brick (physical data) and its logical coordinates
# Brick: Struct containing physical_data[BRICK_PHYSICAL_SIZE^3] and logical_coords[3]

def preprocess_volume(source_data):
    """
    Pre-process source data into a brick pyramid.
    Returns a server-side storage structure of compressed bricks.
    """
    pyramid = []
    current_level = source_data
    while current_level is not None:
        bricks = []
        # Split current level into 64^3 logical chunks
        # Store as 66^3 physical chunks with padding for interpolation
        for chunk in split_into_chunks(current_level, BRICK_LOGICAL_SIZE):
            padded_chunk = pad_to_size(chunk, BRICK_PHYSICAL_SIZE)
            compressed = compress(padded_chunk)
            bricks.append(compressed)
        pyramid.append(bricks)
        # Generate next LOD level by downsampling
        current_level = downsample(current_level, factor=2)
    return pyramid

def get_relevant_bricks(camera, pyramid):
    """
    Identify bricks relevant to the current camera view and determine LOD.
    Returns list of (brick_id, lod_level, logical_coords).
    """
    relevant = []
    # Determine bounding box of camera frustum in logical brick coordinates
    frustum_bbox = camera.frustum_to_brick_coords(pyramid[0].dimensions)
    
    for lod_level, bricks in enumerate(pyramid):
        # Check which bricks in this LOD level intersect the frustum
        for brick_id, brick in enumerate(bricks):
            if intersects_frustum(brick.logical_coords, frustum_bbox):
                relevant.append((brick_id, lod_level, brick.logical_coords))
    return relevant

def render_frame(camera, indirection_texture, cache, server):
    """
    Render a single frame using out-of-core streaming.
    """
    # 1. Identify relevant bricks
    relevant_bricks = get_relevant_bricks(camera, server.pyramid)
    
    # 2. Fetch and decompress asynchronously
    # This step happens in worker threads
    fetched_bricks = []
    for brick_id, lod_level, logical_coords in relevant_bricks:
        compressed_data = server.fetch_range_read(brick_id, lod_level)
        decompressed_data = decompress(compressed_data)
        fetched_bricks.append({
            'logical_coords': logical_coords,
            'data': decompressed_data,
            'lod_level': lod_level
        })
    
    # 3. Update Cache and Indirection
    for brick_info in fetched_bricks:
        logical_coords = brick_info['logical_coords']
        data = brick_info['data']
        
        # Find a cache slot
        slot_index = find_free_or_lru_slot(cache)
        
        # Evict if necessary (LRU)
        if cache[slot_index].is_occupied():
            evict_lru(cache)
        
        # Place brick in cache
        cache[slot_index].data = data
        cache[slot_index].logical_coords = logical_coords
        
        # Update indirection texture
        indirection_texture.set(logical_coords, slot_index)
    
    # 4. Ray Marching
    # For each pixel, cast a ray
    for y in range(image_height):
        for x in range(image_width):
            ray = camera.generate_ray(x, y)
            color = 0.0
            t = ray.origin
            while t < ray.max_distance:
                # Resolve indirection
                # Convert world position t to logical brick coordinates
                brick_coords = world_to_brick_coords(t)
                slot_index = indirection_texture.get(brick_coords)
                
                # Get brick data from cache
                brick = cache[slot_index].data
                
                # Convert world position t to local brick coordinates
                local_coords = world_to_local_brick_coords(t, brick_coords)
                
                # Sample density
                density = sample_density(brick, local_coords)
                
                # Composite
                color = composite(color, density)
                
                # Advance t
                t += ray.step_size
            image[x, y] = color

def find_free_or_lru_slot(cache):
    """
    Find a free slot or the LRU slot.
    """
    for i, slot in enumerate(cache):
        if not slot.is_occupied():
            return i
    # If no free slot, find LRU
    lru_slot = min(cache, key=lambda s: s.last_access_time)
    return cache.index(lru_slot)

def evict_lru(cache):
    """
    Evict the LRU brick from the cache.
    """
    lru_slot = min(cache, key=lambda s: s.last_access_time)
    lru_slot.clear()
```

**Parameters:**
*   **Brick Size:** $64^3$ logical voxels, $66^3$ physical voxels [S1, unofficial].
*   **GPU Cache Size:** 512 slots (default), scalable to $768^3$ or $1024^3$ [S1, unofficial].
*   **Render Scale:** May need reduction (e.g., 0.25) on older/mobile GPUs to maintain 60 FPS [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Texture bandwidth intensity can cause performance drops on older/mobile GPUs [S1, unofficial].
*   **Boundaries:** Larger datasets than tested (~2048³ voxels) require manual adjustment of indirection table dimensions and page cache size [S1, unofficial].
*   **Degenerate Input:** Raw volume files cannot be rendered directly; they must be pre-processed and compressed, adding an offline ingestion step and increasing storage by ~15–20% [S1, unofficial].
*   **Network Dependency:** User experience is tied to internet connection quality; lacks smart pre-fetching or offline capabilities [S1, unofficial].

## Subtopics

### [I/O-Conscious Volume Rendering](6929-i-o-conscious-volume-rendering.md)
A volume rendering technique that hides disk I/O delay associated with large-scale volume data set rendering by overlapping disk accesses with rendering computation, using a data-driven approach to exploit parallelism while minimizing synchronization.

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. Performance depends on network latency, decompression speed, and GPU texture bandwidth [S1, unofficial].
*   **Memory Complexity:** Fixed O(1) VRAM usage determined by cache slot count, independent of the total volumetric dataset size [S1, unofficial].
*   **Measured Performance:**
    *   **Chameleon Dataset (1024×1024×1080, 2160 MB):** First Render on M2: 544 ms, Zenfone: 655 ms; Data Streamed: M2 0.73 MB, Zenfone 0.77 MB; Atlas Occupancy: 2/512 (0.4%) [S1, unofficial].
    *   **Beechnut Dataset (1024×1024×1546, 3092 MB):** First Render on M2: 1079 ms, Zenfone: 2162 ms; Data Streamed: M2 1.10 MB, Zenfone 2.19 MB; Atlas Occupancy: 18/512 (4%) [S1, unofficial].
*   **Scaling:** The architecture supports volumes up to ~2048³ voxels at 64³ brick size. Larger datasets require manual adjustment of indirection table dimensions and page cache size [S1, unofficial].
*   **Strengths:** Fixed VRAM footprint (~300 MB at 512 slots for 16-bit data), interactive frame rates, decoupling of logical and physical memory [S1, unofficial].
*   **Limitations:** Requires pre-processing and compression, network-dependent, no smart pre-fetching, limited to tested dataset sizes, texture bandwidth intensive [S1, unofficial].
*   **When to Use:** Interactive rendering of multi-GB volumetric datasets in browser environments with limited VRAM [S1, unofficial].
*   **When Not to Use:** When raw volume files must be rendered directly without pre-processing, or when offline capabilities are required [S1, unofficial].

## Background
Out-of-core brick-based volume rendering is implemented in the Kiln project, a WebGPU-native out-of-core rendering system for large virtualized volumetric data [S1, unofficial]. It was developed by Martin Panknin and the Distributed Paleoanthropology group and Visualization and MultiMedia Lab, University of Zurich [S1, unofficial]. The project was accepted to the official OME-NGFF tools registry, making it a recognized viewer for real-world OME-Zarr datasets alongside other established tools from this domain [S1, unofficial]. Kiln is integrated into the OME-NGFF ecosystem (v0.2.1+) and tested on Chameleon and Beechnut datasets [S1, unofficial].

## Sources
[S1] Kiln: WebGPU-Native Out-of-Core Volume Rendering for Multi-GB Datasets, https://dev.to/mpanknin/kiln-webgpu-native-out-of-core-volume-rendering-for-multi-gb-datasets-2alb, fetched 2026-10-26 (unofficial)
