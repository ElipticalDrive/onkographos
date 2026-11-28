# Screen-space Subdivision with Virtual Patches

Also called: Buffered Pixel-planes, Virtual Pixel Approach

## Overview
Screen-space subdivision with virtual patches is a parallel rendering scheme that divides the screen into small, disjoint 128x128-pixel regions (patches) processed simultaneously by independent SIMD engines called Renderers [S1, unofficial]. It differs from previous work using a single large computing surface of SIMD processors operating on the entire screen space by using multiple small engines for flexible mapping and cost/performance trading [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of transformed 3D primitives and the screen resolution (e.g., 1280x1024 pixels).
*   **Output:** A rendered image in the Frame Buffer.

**Data Structures:**
*   **Bins:** A collection of lists, one for each patch in the screen space. Each bin stores the indices or pointers to primitives that overlap the corresponding patch region [S1, unofficial].
*   **Backing Store:** Secondary storage (built from VRAMs) used to save pixel color values for completed patches [S1, unofficial].
*   **On-chip Pixel Memory:** Fast SRAM (approx. 200 bits per pixel processor) associated with each Renderer, holding the current patch's Z-buffer and color values [S1, unofficial].
*   **Frame Buffer:** Double-buffered conventional VRAM used for display refresh [S1, unofficial].

**Numbered Steps:**
1.  **Binning:** Sort transformed primitives into bins corresponding to each 128x128 patch-sized region of screen space. Primitives falling into multiple regions are placed into the bins for all such regions [S1, unofficial].
2.  **Assignment:** Assign all available Renderers to patches on the screen. Dispatch primitives from their corresponding bins to these Renderers [S1, unofficial].
3.  **Rendering:** Each Renderer processes its assigned patch.
4.  **Context Switch:** When a Renderer completes a patch, it discards its Z-buffer and all other pixel values besides colors. It transfers pixel color values from on-chip pixel memory to the backing store [S1, unofficial].
5.  **Reassignment:** The Renderer is assigned to the next patch to be processed. Patches are processed in raster order [S1, unofficial].
6.  **Final Transfer:** After the entire image is rendered, transfer each region from the backing store in a block to the double-buffered display memory in the Frame Buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
PATCH_WIDTH = 128
PATCH_HEIGHT = 128
SCREEN_WIDTH = 1280
SCREEN_HEIGHT = 1024
NUM_PATCHES_X = SCREEN_WIDTH // PATCH_WIDTH  # 10
NUM_PATCHES_Y = SCREEN_HEIGHT // PATCH_HEIGHT # 8
TOTAL_PATCHES = NUM_PATCHES_X * NUM_PATCHES_Y # 80

# Data Structures
# bins: List of lists, indexed by patch_id (0 to TOTAL_PATCHES-1)
bins = [[] for _ in range(TOTAL_PATCHES)]
# backing_store: Array of size TOTAL_PATCHES, each element holds color data for a patch
backing_store = [None] * TOTAL_PATCHES
# frame_buffer: Double-buffered display memory
frame_buffer = create_double_buffered_framebuffer(SCREEN_WIDTH, SCREEN_HEIGHT)

def get_patch_id(x, y):
    """Map screen coordinates to patch ID in raster order."""
    px = x // PATCH_WIDTH
    py = y // PATCH_HEIGHT
    return py * NUM_PATCHES_X + px

def bin_primitives(primitives):
    """Sort primitives into bins based on their screen-space bounding boxes."""
    for prim in primitives:
        # Determine the bounding box of the primitive in screen space
        min_x, max_x, min_y, max_y = prim.screen_bounds()
        
        # Determine the range of patch IDs this primitive overlaps
        start_px = max(0, min_x // PATCH_WIDTH)
        end_px = min(NUM_PATCHES_X - 1, max_x // PATCH_WIDTH)
        start_py = max(0, min_y // PATCH_HEIGHT)
        end_py = min(NUM_PATCHES_Y - 1, max_y // PATCH_HEIGHT)
        
        for py in range(start_py, end_py + 1):
            for px in range(start_px, end_px + 1):
                patch_id = py * NUM_PATCHES_X + px
                bins[patch_id].append(prim)

def renderer_work(patch_id):
    """Process a single patch."""
    # Load primitives for this patch
    primitives = bins[patch_id]
    
    # Initialize on-chip memory for this patch
    # Z-buffer and color buffer for 128x128 pixels
    z_buffer = initialize_z_buffer(PATCH_WIDTH, PATCH_HEIGHT)
    color_buffer = initialize_color_buffer(PATCH_WIDTH, PATCH_HEIGHT)
    
    # Render primitives
    for prim in primitives:
        rasterize(prim, z_buffer, color_buffer)
        
    # Save results to backing store
    backing_store[patch_id] = color_buffer
    
    # Discard Z-buffer and on-chip color buffer to free resources
    # (Implementation detail: memory is reused for next patch)

def render_scene(primitives, num_renderers):
    bin_primitives(primitives)
    
    # Queue of patches to process, in raster order
    patch_queue = list(range(TOTAL_PATCHES))
    
    # Assign initial patches to renderers
    active_renderers = []
    for i in range(min(num_renderers, len(patch_queue))):
        patch_id = patch_queue.pop(0)
        renderer_work(patch_id)
        active_renderers.append(patch_id)
        
    # Process remaining patches as renderers become free
    while patch_queue:
        # Wait for any renderer to finish (simplified simulation)
        # In hardware, this is handled by the controller
        completed_patch = active_renderers.pop(0)
        
        if patch_queue:
            next_patch = patch_queue.pop(0)
            renderer_work(next_patch)
            active_renderers.append(next_patch)
            
    # Final transfer to frame buffer
    for patch_id in range(TOTAL_PATCHES):
        patch_color_data = backing_store[patch_id]
        # Calculate screen coordinates for the patch
        py = patch_id // NUM_PATCHES_X
        px = patch_id % NUM_PATCHES_X
        
        # Transfer block to frame buffer
        transfer_to_framebuffer(
            frame_buffer,
            patch_color_data,
            x_offset=px * PATCH_WIDTH,
            y_offset=py * PATCH_HEIGHT,
            width=PATCH_WIDTH,
            height=PATCH_HEIGHT
        )

# Typical Parameters
# Patch size: 128x128 pixels [S1, unofficial]
# Display resolution: 1280x1024 pixels [S1, unofficial]
# Number of patches: 80 [S1, unofficial]
```

**Edge Cases and Pitfalls:**
*   **Primitive Overlap:** Primitives falling into more than one region must be processed multiple times. This reduces actual performance compared to raw performance [S1, unofficial].
*   **Context Switch Overhead:** Switching between patches incurs a context switch time of ~0.4 msec, which is the time to render about 100 primitives. This overhead must be overlapped with pixel processing to maintain efficiency [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering into a window of arbitrary size incurs only a linear time penalty [S1, unofficial].
*   **Performance Metrics:**
    *   Single Renderer raw performance: ~150,000 Phong-shaded triangles per second [S1, unofficial].
    *   Single Renderer simulated actual performance: ~100,000 triangles/sec (reduced due to multi-patch processing) [S1, unofficial].
    *   Estimated Graphics Processor (GP) performance: ~30,000 Phong-shaded triangles per second [S1, unofficial].
*   **Scalability:** The system allows cost/performance to be traded nearly linearly by adding or removing small rendering units [S1, unofficial].
*   **Strengths:** Modular, flexible mapping of virtual pixel spaces, efficient use of multiple independent SIMD engines.
*   **Limitations:** Performance reduction due to redundant processing of primitives spanning multiple patches; context switch overhead.

## Background
This scheme was developed for the PxplS system, a heterogeneous multiprocessor graphics system. It builds on previous work using "buffered" Pixel-planes but introduces the concept of flexibly mapping small virtual pixel spaces onto the screen space [S1, unofficial]. The design aims to fit within a workstation cabinet with no unusual power requirements, drive a 1280x1024 display at >60Hz, and update full scene images at >20 frames/second [S1, unofficial].

## Sources
[S1] DTIC ADA208089: A Heterogeneous Multiprocessor Graphics System Using Processor-Enhanced Memories, https://archive.org/details/DTIC_ADA208089, fetched 2026-10-26 (unofficial)
