# Virtual Texture Management with Bricking

Also called: Custom Bricking, TMRenderAction Texture Padding and Virtualization, Texture Management

## Overview
Virtual Texture Management with Bricking handles textures larger than available texture memory by virtually partitioning them into bricks, managing their loading and sorting transparently to prevent fragmentation [S1]. It sets itself apart from its siblings by automating the padding of non-power-of-two dimensions and the virtualization of oversized textures without requiring manual brick creation or management from the application layer [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume texture with arbitrary dimensions and size, potentially exceeding GPU texture memory limits.
*   **Output:** A valid OpenGL texture representation, potentially split into multiple bricks, resident in texture memory as needed.

**Data Structures:**
*   **Texture Record:** For each brick, a record stores the quality and delivery state held by that brick. This record mirrors the quality record of the composite, allowing comparison to identify out-of-date bricks [S2, unofficial].
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
# Pseudocode based on TMRenderAction behavior [S1]

def process_texture(texture_data, texture_dims, available_memory):
    """
    Processes a volume texture for rendering, handling padding and bricking.
    """
    # Step 1: Validate and pad dimensions
    padded_dims = []
    for dim in texture_dims:
        # Pad to next power of two
        next_pow2 = 1
        while next_pow2 < dim:
            next_pow2 *= 2
        padded_dims.append(next_pow2)
    
    # Pad texture data to new dimensions
    padded_texture_data = pad_texture_data(texture_data, texture_dims, padded_dims)
    
    # Step 2: Check size against available memory
    total_size = product(padded_dims)
    
    if total_size <= available_memory:
        # Create single OpenGL texture
        gl_texture = create_gl_texture(padded_dims, padded_texture_data)
        return [gl_texture]
    else:
        # Step 3: Break into bricks
        brick_dims = get_default_brick_dims() # e.g., 64x64x64
        bricks = []
        
        # Iterate through the padded volume in brick-sized chunks
        for z in range(0, padded_dims[2], brick_dims[2]):
            for y in range(0, padded_dims[1], brick_dims[1]):
                for x in range(0, padded_dims[0], brick_dims[0]):
                    # Extract brick data
                    brick_data = extract_brick(padded_texture_data, x, y, z, brick_dims)
                    # Create OpenGL texture for this brick
                    gl_brick = create_gl_texture(brick_dims, brick_data)
                    bricks.append(gl_brick)
        
        return bricks

def pad_texture_data(data, original_dims, padded_dims):
    """
    Pads texture data to the specified dimensions.
    """
    # Implementation detail: [G]
    # This involves allocating a new buffer of size padded_dims and copying
    # original data into the appropriate offsets, filling the rest with
    # appropriate padding values (e.g., 0 or edge values).
    pass

def extract_brick(data, x, y, z, brick_dims):
    """
    Extracts a sub-volume from the texture data.
    """
    # Implementation detail: [G]
    # This involves calculating the memory offset for the brick and copying
    # the relevant voxel data.
    pass

def get_default_brick_dims():
    """
    Returns default brick dimensions.
    """
    return [64, 64, 64] # Example default

def product(dimensions):
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

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Transparent management of brick creation, management, and sorting; handles arbitrary texture dimensions and sizes; prevents fragmentation [S1].
*   **Limitations:** Relies on `TMRenderAction` for automation; may introduce overhead from padding and bricking if not managed carefully [S1].
*   **When to Use:** When textures exceed available texture memory or have non-power-of-two dimensions [S1].
*   **When Not to Use:** When custom bricking is required for specific performance optimizations [S1].

## Background
Originated in the SGI Virtual Laboratory (vz) library, specifically within the `TMRenderAction` component [S1]. It was designed to simplify volume rendering by abstracting the complexities of texture memory management [S1]. The technique is part of the broader volume rendering pipeline, where it handles the "Texture Management" phase [S1].

## Sources
[S1] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
[S2] Volume textures: brick copies in the render, a recorded update copy, paced renders, and a preload that states its loader, https://github.com/cornerstonejs/cornerstone3D/issues/2966, fetched 2026-10-27 (unofficial)
