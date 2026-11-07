# Demand-Driven Volume Rendering

Also called: Demand-Driven Data Back-End, Virtual Octree Rendering

## Overview
Demand-Driven Volume Rendering is a GPU ray-casting technique that renders a "virtual octree" where physical voxel data is computed on demand, driven by visibility and screen-projection size, allowing interactive rendering of terascale electron microscopy (EM) data without pre-computing a 3D multi-resolution hierarchy [S1]. It differs from sibling methods like GigaVoxels by decoupling the ray-caster from the data reconstruction backend, using a two-level page table hierarchy to translate virtual sample positions to physical data on the fly, enabling interactive frame rates for volumes exceeding GPU memory [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A stream of 2D image tiles (potentially of different resolution, scale, and orientation) representing the source volume data [S1].
- **Output:** A rendered 2D image frame [S1].

**Data Structures:**
The system relies on a multi-level paged virtual memory management system consisting of three cache levels [S1]:
1.  **Top-level Page Directory:** Fully resident in GPU memory. For a 1TB volume, this is 48 KB [S1].
2.  **2nd-level Page Table Cache:** Resides in a small cache (GPU or CPU). For a 1TB volume, this is 32 MB [S1]. Entries refer to physical octree nodes [S1].
3.  **Physical Texture Cache:** Contains physical octree nodes. Size ranges from 1 GB (single GPU) to several gigabytes (multi-GPU) [S1].

**Octree Node Layout:**
- Each octree node contains $32^3$ physical voxels [S1].
- Nodes are addressed via the page table hierarchy [S1].

**Parameters:**
- Octree node size: $32^3$ voxels [S1].
- Top-level page directory size: 48 KB (for 1TB volume) [S1].
- 2nd-level page table cache size: 32 MB (for 1TB volume) [S1].
- Physical texture cache size: 1 GB to several GB [S1].

**Algorithm Steps:**
1.  **Initialization:** Load the top-level page directory into GPU memory [S1]. Initialize the 2nd-level page table cache and physical texture cache [S1]. Start the demand-driven data back-end in a separate background thread [S1].
2.  **Ray Casting Loop (per frame):**
    a. For each pixel, cast a ray into the virtual volume space [S1].
    b. Compute sample positions in virtual volume space [S1].
    c. Translate virtual sample positions to physical addresses using the two-level page table hierarchy:
        i. Look up the top-level page directory (resident in GPU) to find the relevant 2nd-level page table [S1].
        ii. Look up the 2nd-level page table cache to find the physical octree node address [S1].
    d. Check if the physical octree node is resident in the Physical Texture Cache [S1].
    e. **If resident:** Retrieve the node data and perform front-to-back volume rendering [S1].
    f. **If not resident (Cache Miss):**
        i. Generate a cache miss signal [S1].
        ii. Substitute missing data with lower resolution data to maintain interactive frame rates [S1].
        iii. Trigger the demand-driven data back-end to reconstruct the physical data of matching resolution from the stream of 2D image tiles [S1].
        iv. Continue rendering with lower resolution data while reconstruction proceeds in the background [S1].
3.  **Data Reconstruction (Background Thread):**
    a. Receive requests for physical octree nodes from the ray-caster [S1].
    b. If the required 2D image tiles are in memory, reconstruct the 3D block ($512 \times 512 \times 32$ or similar) as fast as 30ms [S1].
    c. If 2D image tiles are not in memory, read them from disk (latency up to a few seconds) [S1].
    d. Populate the Physical Texture Cache with the reconstructed nodes [S1].
    e. Update the 2nd-level page table cache entries to point to the new physical nodes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global State
top_level_page_directory: List[PageTableEntry] # Resident in GPU, size 48KB for 1TB vol
page_table_cache: Dict[VirtualAddress, PhysicalAddress] # Size 32MB for 1TB vol
physical_texture_cache: Dict[PhysicalAddress, OctreeNode] # Size 1GB+
octree_node_size = 32 # Voxels per dimension

# Background Thread: Demand-Driven Data Back-End
def reconstruction_thread():
    while True:
        request = get_reconstruction_request() # From ray-caster
        if request is None:
            continue
        
        # Reconstruct physical data from 2D image tiles
        # Latency: ~30ms if tiles in memory, up to seconds if from disk
        physical_data = reconstruct_from_2d_tiles(request.virtual_address)
        
        # Create octree node
        node = OctreeNode(physical_data)
        
        # Update caches
        physical_address = request.physical_address
        physical_texture_cache[physical_address] = node
        page_table_cache[request.virtual_address] = physical_address

# Ray-Caster
def render_frame():
    for each pixel:
        ray = cast_ray(pixel)
        color = 0.0
        
        while ray_has_samples(ray):
            sample_pos = ray.get_next_sample_position()
            virtual_address = virtual_to_virtual_address(sample_pos)
            
            # Two-level page table lookup
            if virtual_address in page_table_cache:
                physical_address = page_table_cache[virtual_address]
            else:
                # Look up in top-level directory to find 2nd-level table
                # (Simplified: assume top-level directory maps to 2nd-level table pointers)
                # If 2nd-level table not in cache, it must be loaded (handled by cache miss logic)
                physical_address = None
            
            if physical_address is not None and physical_address in physical_texture_cache:
                node = physical_texture_cache[physical_address]
                color += ray_march(ray, node)
            else:
                # Cache Miss
                # Substitute with lower resolution data
                color += ray_march_low_res(ray)
                
                # Trigger reconstruction
                trigger_reconstruction(virtual_address)
        
        set_pixel_color(pixel, color)

def trigger_reconstruction(virtual_address):
    # Send request to background thread
    send_to_back_end(virtual_address)
```

**Edge Cases and Pitfalls:**
- **Reconstruction Latency:** If 2D image tiles are not in memory, reconstruction may take up to a few seconds [S1]. This is handled by substituting missing data with lower resolution data while reconstruction proceeds in the background [S1].
- **Spatial Coherence:** Performance relies on spatial coherence along rays to make page table lookups fast [S1].
- **Cache Size Scaling:** The system scales to very large volumes because the top-level page directory is small (48 KB for 1TB) and the working set is bounded by display resolution, not volume resolution [S1].

## Operating characteristics
- **Time Complexity:** Page table lookups are extremely fast when spatial coherence is exploited [S1]. Reconstruction latency varies from 30ms (tiles in memory) to seconds (tiles from disk) [S1].
- **Memory Complexity:** Top-level page directory: 48 KB for 1TB volume [S1]. 2nd-level page table cache: 32 MB for 1TB volume [S1]. Physical texture cache: 1 GB to several GB [S1].
- **Measured Performance:** Interactive frame rates for volumes up to 1TB [S1]. Mouse cortex volume (955 GB) rendered interactively [S1].
- **Strengths:** Interactive rendering of terascale data without pre-computation; decoupled backend allows progressive reconstruction; small memory footprint for page tables [S1].
- **Limitations:** Latency in data reconstruction if tiles are not in memory; requires a stream of 2D image tiles [S1].
- **When to use:** Interactive exploration of very large volumes (e.g., EM data) where pre-computation is infeasible or too slow [S1].
- **When not to use:** When low latency is critical and tiles are not pre-loaded; when the volume is small enough to fit in GPU memory without paging [S1].

## Background
Demand-Driven Volume Rendering was introduced in 2011 by researchers from Harvard University (SIGGRAPH 2011) [S1]. It was developed to enable 3D volume rendering of large electron microscopy (EM) datasets, such as the mouse cortex (955 GB) and hippocampus (43 GB, 92 GB) volumes acquired by the Harvard Center for Brain Science [S1]. The method builds on the concept of virtual octrees but differs from prior systems like GigaVoxels by using a simpler, faster scheme for volume traversal based on arbitrary direct access to the full virtual octree in virtual memory, driven by a decoupled data back-end [S1].

## Sources
[S1] Demand-Driven Volume Rendering of Terascale EM Data, https://vcg.seas.harvard.edu/publications/20110807-demand-driven-volume-rendering-of-terascale-em-data/paper, fetched 2026-10-28
