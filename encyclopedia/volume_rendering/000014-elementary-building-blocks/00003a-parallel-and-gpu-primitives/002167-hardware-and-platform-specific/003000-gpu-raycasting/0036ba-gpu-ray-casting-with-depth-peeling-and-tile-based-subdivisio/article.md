# GPU Ray Casting with Depth Peeling and Tile-Based Subdivision

Also called: Bernardon et al. GPU Ray Casting, Bernardon et al, Screen Tiling for Depth Peeling, Tile-based depth peeling, Weiler et al. Depth Peeling, Weiler et al

## Overview
This method improves GPU-based ray casting for unstructured grids by combining tile-based screen subdivision with depth peeling to handle non-convex meshes efficiently [S2, S3, unofficial]. It solves the problem of "imaginary cells" caused by mesh non-convexities without requiring convexification, while reducing fragment processing overhead by restricting computation to relevant screen regions [S2, S3, unofficial]. It differs from standard GPU ray casting by explicitly managing visibility layers via depth peeling per tile, and from convexification-based approaches by capturing ray re-entry into the mesh naturally [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Unstructured mesh data (vertices, connectivity), viewing parameters (camera position, direction).
- Output: Rendered image with correct visibility for non-convex geometry.

**Data Structures:**
- **Mesh Representation:** Stored in 2D textures for compactness and efficiency, replacing the 3D textures used in earlier works [S2, S3, unofficial]. The exact layout of the 2D texture is not specified in the sources [G: Implementers should map mesh attributes (e.g., vertex positions, cell indices) into a 2D texture atlas or structured 2D grid based on mesh topology].
- **Screen Tiles:** The screen is subdivided into uniform square tiles.
- **Intersection Data:** For each tile, a list of intersecting primitives (e.g., spheres in metaball contexts, or mesh cells) is maintained [S1].

**Steps:**
1. **Tile Subdivision:** Divide the screen into uniform tiles. Typical tile sizes are 8x8 pixels for a 640x480 resolution and 16x16 pixels for a 1024x768 resolution [S1].
2. **Culling/Registration:** For each tile, determine which primitives (e.g., spheres or mesh cells) intersect the tile's frustum or bounding volume [S1]. Register these primitives for rendering within that tile [S1].
3. **Depth Peeling per Tile:** Perform depth peeling for each tile independently [S1].
    - **Layer 1:** Render the closest intersection for each ray within the tile.
    - **Subsequent Layers:** For each subsequent layer, render only the intersections that are farther away than the previous layer's depth [S1].
    - **Termination:** Depth peeling continues until no more intersections are found or a maximum number of layers is reached. Termination can be automatically detected using occlusion queries [S1].
4. **Ray Casting:** For each fragment in the active tiles, perform the ray-primitive intersection test in the fragment shader [S1]. Discard fragments with no intersection [S1].
5. **Rendering:** Render either front or back faces of the primitives, depending on the depth peeling layer and visibility requirements [S1].

**Formulas:**
- Not given in the sources.

**Parameters:**
- Tile size: 8x8 pixels for 640x480 resolution; 16x16 pixels for 1024x768 resolution [S1].

**Edge Cases and Pitfalls:**
- **Non-convex Meshes:** The method avoids introducing imaginary cells that fill space due to non-convexities by using depth peeling to capture ray re-entry [S2, S3, unofficial].
- **Occlusion Query Overhead:** Using occlusion queries for termination detection may introduce CPU-GPU synchronization overhead [S1].
- **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GPU Ray Casting with Depth Peeling and Tile-Based Subdivision
# Note: This is a high-level representation. Actual implementation requires GPU shaders and API calls.

def render_with_tile_based_depth_peeling(mesh, camera, screen_width, screen_height):
    # 1. Determine tile size based on resolution
    if screen_width == 640 and screen_height == 480:
        tile_size = 8
    elif screen_width == 1024 and screen_height == 768:
        tile_size = 16
    else:
        tile_size = 8 # Default or calculate based on resolution [G]

    # 2. Subdivide screen into tiles
    tiles = create_tiles(screen_width, screen_height, tile_size)

    # 3. For each tile, cull and register intersecting primitives
    for tile in tiles:
        intersecting_primitives = cull_primitives_for_tile(mesh, tile, camera)
        register_primitives_for_tile(tile, intersecting_primitives)

    # 4. Perform depth peeling for each tile
    for tile in tiles:
        if not tile.has_intersecting_primitives():
            continue

        # Initialize depth peeling layers
        current_depth = -infinity
        layer = 0
        max_layers = 100 # Arbitrary limit [G]

        while layer < max_layers:
            # Render current layer for the tile
            # This involves fragment shader intersection tests
            # and depth comparison against current_depth
            rendered_primitives = render_depth_peel_layer(tile, current_depth, camera)

            if not rendered_primitives:
                break # No more intersections, terminate depth peeling

            # Update current_depth to the maximum depth of the current layer
            current_depth = max_depth(rendered_primitives)
            layer += 1

            # Optional: Use occlusion query to detect termination
            # if is_occluded(tile):
            #     break

    # 5. Composite layers to produce final image
    composite_image = composite_depth_peeling_layers(tiles)
    return composite_image

def cull_primitives_for_tile(mesh, tile, camera):
    # Determine which primitives intersect the tile's view frustum
    # This can be done using bounding volume hierarchy or simple bounding box tests
    intersecting = []
    for primitive in mesh.primitives:
        if intersects_frustum(primitive, tile, camera):
            intersecting.append(primitive)
    return intersecting

def render_depth_peel_layer(tile, current_depth, camera):
    # Fragment shader logic for a single depth peeling layer
    # For each fragment in the tile:
    #   1. Generate ray from camera through fragment
    #   2. Intersect ray with registered primitives
    #   3. Find closest intersection with depth > current_depth
    #   4. If found, render primitive; else, discard fragment
    # Return list of rendered primitives for this layer
    pass

def composite_depth_peeling_layers(tiles):
    # Combine all depth peeling layers from all tiles
    # This may involve alpha blending or other compositing techniques
    pass
```

## Operating characteristics
- **Time Complexity:** Depends on the number of tiles, primitives per tile, and depth peeling layers. Tile-based subdivision reduces the number of fragments processed by culling empty regions [S2, S3, unofficial].
- **Memory Complexity:** 2D texture representation is more compact than 3D textures [S2, S3, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with screen resolution and mesh complexity. Larger tiles may reduce overhead but increase fragment processing per tile [S1].
- **Strengths:** Handles non-convex meshes without convexification; reduces fragment processing via tiling [S2, S3, unofficial].
- **Limitations:** Depth peeling can be computationally expensive for complex scenes with many layers; occlusion query overhead [S1].
- **When to Use:** Suitable for rendering non-convex unstructured grids on GPU where convexification is difficult or undesirable [S2, S3, unofficial].
- **When Not to Use:** Not suitable for simple convex meshes where standard ray casting suffices; may be inefficient for scenes with very few primitives per tile [G].

## Background
- **Origin:** Proposed by Bernardon et al. [5] and concurrently by Weiler et al. [67] [S2, S3, unofficial].
- **History:** Developed as an improvement over earlier GPU-based ray casting methods that used 3D textures and convexification [S2, S3, unofficial].
- **Usage:** Used in GPU-based volume rendering of unstructured grids [S2, S3, unofficial].

## Sources
[S1] GPU‐based Fast Ray Casting for a Large Number of Metaballs, https://doi.org/10.1111/j.1467-8659.2008.01132.x, fetched 2026-11-01
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S3] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
