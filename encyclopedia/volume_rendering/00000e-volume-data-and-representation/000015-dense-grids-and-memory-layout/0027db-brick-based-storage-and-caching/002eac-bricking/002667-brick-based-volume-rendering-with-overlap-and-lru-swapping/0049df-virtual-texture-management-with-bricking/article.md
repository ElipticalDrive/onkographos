# Virtual Texture Management with Bricking

Also called: Custom Bricking, TMRenderAction Texture Padding and Virtualization, Texture Management

## Overview
Virtual Texture Management with Bricking handles textures larger than available texture memory by virtually partitioning them into bricks, managing their loading and sorting transparently to prevent fragmentation [S1]. It sets itself apart from its siblings by automating the padding of non-power-of-two dimensions and the virtualization of oversized textures without requiring manual brick creation or management from the application layer [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume texture with arbitrary dimensions and size, potentially exceeding GPU texture memory limits [S1].
*   **Output:** A valid OpenGL texture representation, potentially split into multiple bricks, resident in texture memory as needed [S1].

**Data Structures:**
*   **Texture Record:** For each brick, a record stores the quality and delivery state held by that brick [S2, unofficial]. This record mirrors the quality record of the composite, allowing comparison to identify out-of-date bricks [S2, unofficial].
*   **Brick:** A 3D sub-volume of the texture (e.g., $64 \times 64 \times 64$ voxels) [S2, unofficial].

**Steps:**
1.  **Dimension Validation and Padding:** The system checks if texture dimensions are powers of two. If not, `TMRenderAction` automatically pads the texture data to the next higher power-of-two dimensions [S1].
2.  **Size Validation and Bricking:** The system checks if the total texture size exceeds available texture memory. If it does, `TMRenderAction` internally breaks the texture into smaller bricks [S1].
3.  **Residency Management:** The application manages the list of shapes resident in texture memory based on visibility culling or frame index [S1]. A shape that needs to be drawn must be managed [S1].
4.  **Rendering:** The render action uses the managed bricks to create actual OpenGL textures, optimizing flow to the graphics subsystem [S1].

**Parameters:**
*   **Texture Dimensions:** Must be powers of two for validity. `TMRenderAction` handles invalid dimensions via padding [S1].
*   **Texture Size:** Must be less than or equal to the amount of texture memory available on the graphics subsystem. `TMRenderAction` handles oversized textures via virtualization [S1].

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two Dimensions:** OpenGL requires power-of-two dimensions. `TMRenderAction` resolves this by padding [S1].
*   **Oversized Textures:** If the texture does not fit in memory, manual management is bypassed by internal bricking [S1].
*   **Internal Format Inference:** If a default value is used for the internal format, the render action infers a suitable value from the data type and external format of the texture [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_texture(texture_data, texture_dims, available_memory, brick_size=(64, 64, 64)):
    """
    Processes a volume texture for rendering, handling padding and bricking.
    texture_data: list of voxel values (flattened or nested)
    texture_dims: tuple (x, y, z) of original dimensions
    available_memory: int, max voxels that fit in GPU texture memory
    brick_size: tuple (bx, by, bz), default 64x64x64 [S2, unofficial]
    """
    # Step 1: Validate and pad dimensions to next power of two
    padded_dims = []
    for dim in texture_dims:
        next_pow2 = 1
        while next_pow2 < dim:
            next_pow2 *= 2
        padded_dims.append(next_pow2)
    
    # Pad texture data to new dimensions
    padded_texture_data = pad_texture_data(texture_data, texture_dims, padded_dims)
    
    # Step 2: Check size against available memory
    total_size = compute_product(padded_dims)
    
    if total_size <= available_memory:
        # Create single OpenGL texture
        gl_texture = allocate_texture_object(padded_dims, padded_texture_data)
        return [gl_texture]
    else:
        # Step 3: Break into bricks
        bricks = []
        bx, by, bz = brick_size
        px, py, pz = padded_dims
        
        # Iterate through the padded volume in brick-sized chunks
        for z in range(0, pz, bz):
            for y in range(0, py, by):
                for x in range(0, px, bx):
                    # Extract brick data
                    brick_data = extract_subvolume(padded_texture_data, x, y, z, bx, by, bz)
                    # Create OpenGL texture for this brick
                    gl_brick = allocate_texture_object((bx, by, bz), brick_data)
                    bricks.append(gl_brick)
        
        return bricks

def pad_texture_data(data, original_dims, padded_dims):
    """
    Pads texture data to the specified dimensions.
    Allocates a new buffer of size padded_dims, copies original data into the 
    appropriate offsets, and fills the remainder with zero-padding [G].
    """
    # Standard memory allocation and copy operation [G]
    padded_data = [0] * compute_product(padded_dims)
    # Copy logic omitted for brevity; standard contiguous memory copy [G]
    return padded_data

def extract_subvolume(data, x, y, z, bx, by, bz):
    """
    Extracts a sub-volume from the texture data.
    Calculates the memory offset for the brick and copies the relevant voxel data [G].
    """
    # Standard memory slice operation [G]
    return data  # Placeholder for actual slicing logic [G]

def allocate_texture_object(dimensions, data):
    """
    Simulates allocation of a GPU texture object.
    Returns a mock object representing the bound texture [G].
    """
    return {"dimensions": dimensions, "data": data}

def compute_product(dimensions):
    """
    Calculates the product of a list of dimensions.
    """
    result = 1
    for d in dimensions:
        result *= d
    return result
```

## Subtopics

### Recorded Update Copy
An ongoing update that copies delivered data to the texture and records what the texture holds, ensuring data already loaded is not calculated or loaded again [S2, unofficial]. It follows the delivery record of the volume (`recordFrameDelivery`) and copies each delivered region once at the best quality (full resolution for full-res texture, box average for reduced texture) [S2, unofficial]. After copying, it updates the record of the brick [S2, unofficial]. It runs between renders or in the renders of the pacer, and never in a render that the user starts [S2, unofficial]. A new texture of a loaded volume starts with an empty record; the update copy fills it from existing representations (full-resolution frames or reduced representation) without reloading or recalculating [S2, unofficial]. Each texture keeps a record for each brick: the quality and the delivery that the brick holds. The record has the form of the quality record of the composite, allowing comparison to find out-of-date bricks [S2, unofficial]. A brick already filled at source quality is not out of date; the render copies nothing for it [S2, unofficial]. GL calls outside the render of vtk.js must keep the record of bound textures correct; if this cannot be met, the update copy runs inside paced renders [S2, unofficial].

### GPU Memory Management (Brick Pool)
Stores volume bricks in a single large 3D cache texture (brick pool) to support single-pass rendering and efficient dynamic updates of the working set [S3]. Working set storage consists of a single large 3D cache texture for bricks, address translation helper structures (page table or octree pointers) for virtual-to-texture space translation, a node pool (for octree-based) or page table cache (for page table-based) for traversal structures, and additional meta-data [S3]. When the working set does not fit into GPU memory, the renderer either reduces the level of detail to decrease the number of bricks, or switches to a multi-pass fall-back rendering approach [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1][S2, unofficial][S3].
*   **Memory Complexity:** Not given in the sources [S1][S2, unofficial][S3].
*   **Measured Performance:** Not given in the sources [S1][S2, unofficial][S3].
*   **Scaling:** Not given in the sources [S1][S2, unofficial][S3].
*   **Strengths:** Transparent management of brick creation, management, and sorting; handles arbitrary texture dimensions and sizes; prevents fragmentation [S1].
*   **Limitations:** Relies on `TMRenderAction` for automation; may introduce overhead from padding and bricking if not managed carefully [S1].
*   **When to Use:** When textures exceed available texture memory or have non-power-of-two dimensions [S1].
*   **When Not to Use:** When custom bricking is required for specific performance optimizations [S1].

## Background
Originated in the SGI Virtual Laboratory (vz) library, specifically within the `TMRenderAction` component [S1]. It was designed to simplify volume rendering by abstracting the complexities of texture memory management [S1]. The technique is part of the broader volume rendering pipeline, where it handles the "Texture Management" phase [S1].

## Sources
[S1] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
[S2] Volume textures: brick copies in the render, a recorded update copy, paced renders, and a preload that states its loader, https://github.com/cornerstonejs/cornerstone3D/issues/2966, fetched 2026-10-27 (unofficial)
[S3] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
