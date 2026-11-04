# PND3D Octree Rendering

Also called: PND3D

## Overview
PND3D is a volume rendering method that uses a standard 8-way octree with non-leaf nodes containing child and solid masks, visited in front-to-back depth-first order to enable occlusion culling [S1, unofficial]. It distinguishes itself by splitting the screen into large rectangular tiles determined by 3 vanishing points of the voxel grid projected to screen coordinates, allowing each tile to be rendered as an independent job starting from the root of the octree [S1, unofficial]. This approach facilitates multithread optimization and specific hardware-accelerated occlusion queries using bit arrays [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sparse voxel octree where non-leaf nodes are 64-bit structures.
- Input: Screen resolution and camera parameters (to determine vanishing points).
- Output: A rendered image.
- Intermediate Output: A 64-bit pixel buffer containing position and leaf node index.

**Data Structures:**
Non-leaf nodes are packed 64-bit structures with the following layout [S1, unofficial]:
- `chi` (char): Child byte mask. 1 indicates the presence of children, 0 indicates pure air or solid.
- `sol` (char): Solid byte mask. 1 indicates pure solid, 0 indicates not pure solid.
- `mrk` (char): Alignment padding.
- `mrk2` (char): Alignment padding.
- `ind` (int): Index pointer to the first child.

**Parameters:**
- Occlusion query threshold: 32 pixels width. Nodes with projected bounding box width larger than this are assumed visible. Nodes with width $\le$ 32 pixels use bit array culling [S1, unofficial].
- Tile count: Typically 16-48 tiles per screen [S1, unofficial].
- Pixel format: 64-bit, composed of 12 bits for x, 12 bits for y, 12 bits for z, and a 28-bit index to the leaf node holding surface voxel information [S1, unofficial].

**Steps:**
1. **Tile Division:** Divide the screen into large rectangular tiles. The tile boundaries are determined by the 3 vanishing points of the voxel grid projected to screen coordinates [S1, unofficial].
2. **Bounding Box Calculation:** For each tile, calculate the bounding box of the cubes more quickly by using only 4 of the 8 vertices of a cube, which are known in advance based on the tile's position relative to the vanishing points [S1, unofficial].
3. **Tile Rendering Job:** Render each tile as an independent job, clipped to the tile like a viewport, starting from the root of the octree. This allows for multithread optimization [S1, unofficial].
4. **Occlusion Query:** For every non-leaf node containing some solid, perform an occlusion query on its projected bounding box [S1, unofficial].
   - If the width is larger than 32 pixels, assume it is visible and visit its children [S1, unofficial].
   - If the bounding box width is 32 pixels or less, use a bit array (cover buffer) with 1 bit per pixel to test if the region is fully occluded using an 'and' instruction. If fully occluded, skip the node and its children [S1, unofficial].
5. **Leaf Node Processing:** For leaf nodes (solid surface voxels), visit each pixel of the bounding rectangle and perform 6 dot products in the inner loop using 2 'addps' and 2 'movmskps' instructions to determine if the pixel falls inside the projected cube (a 4- or 6-sided convex polygon) [S1, unofficial].
6. **First Pass (CPU):** Write a 64-bit pixel to the screen containing 12 bits x, 12 bits y, 12 bits z, and a 28-bit index to the leaf node holding surface voxel information. This is done in a first pass on the CPU [S1, unofficial].
7. **Second Pass (GPU):** The GPU receives the voxel position and a pointer to its surface structure, raytraces to the known cube, determines the face, and performs texture mapping with filters like bilinear and mip mapping [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_tile(tile_id, screen_width, screen_height, root_node):
    # Determine tile boundaries based on vanishing points
    tile_bounds = get_tile_bounds(tile_id, screen_width, screen_height)
    
    # Calculate bounding box using 4 known vertices
    bbox = calculate_bbox_4_vertices(tile_bounds)
    
    # Start traversal from root
    traverse_node(root_node, tile_bounds, bbox)

def traverse_node(node, tile_bounds, bbox):
    # Check if node contains solid
    if not node.is_solid():
        return
    
    # Occlusion query
    if bbox.width > 32:
        # Assume visible
        visit_children(node, tile_bounds)
    else:
        # Use bit array culling
        if is_fully_occluded(node, tile_bounds):
            return
        visit_children(node, tile_bounds)

def visit_children(node, tile_bounds):
    for child in node.children:
        child_bbox = calculate_child_bbox(child, tile_bounds)
        traverse_node(child, tile_bounds, child_bbox)

def process_leaf_node(leaf, tile_bounds):
    # Visit each pixel in bounding rectangle
    for pixel in leaf.bounding_rectangle_pixels:
        # Perform 6 dot products to check if pixel is inside projected cube
        if is_pixel_inside_cube(pixel, leaf):
            # Write 64-bit pixel: 12 bits x, 12 bits y, 12 bits z, 28-bit leaf index
            write_pixel(pixel.x, pixel.y, leaf.x, leaf.y, leaf.z, leaf.index)

def is_fully_occluded(node, tile_bounds):
    # Use bit array with 1 bit per pixel
    cover_buffer = get_cover_buffer(tile_bounds)
    # Test if region is fully occluded using 'and' instruction
    return cover_buffer.is_fully_zero()
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Multithread optimization through independent tile rendering; efficient occlusion culling using bit arrays for small bounding boxes; fast bounding box calculation using 4 vertices [S1, unofficial].
- **Limitations:** Requires specific hardware instructions ('addps', 'movmskps') for leaf node processing; may need override for older ATI or Intel video cards [S1, unofficial].
- **When to use:** When multithread optimization is desired and hardware supports the required instructions.
- **When not to use:** When using older ATI or Intel video cards without override, or when highest quality texture-mapping (trilinear filtering) is required on CPU-based emulation [S1, unofficial].

## Background
PND3D was developed by Ken Silverman [S1, unofficial]. It is used for volume rendering with sparse voxel octrees [S1, unofficial]. The engine is available under a non-commercial license [S1, unofficial].

## Sources
[S1] gpdaniels/pnd3d, https://github.com/gpdaniels/pnd3d, fetched 2026-10-27 (unofficial)
