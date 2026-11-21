# Stack-Based Terrain Representation (SBRT)
Also called: Enfoque 1, Representación de Terrenos Basada en Stacks

## Overview
Stack-Based Terrain Representation (SBRT) is a compact 3D data structure that generalizes heightmaps by storing a vertical sequence of intervals for each $(x,y)$ coordinate, where each interval consists of a start height and an attribute (e.g., material type) [S1]. It solves the high memory footprint of voxel-based representations and the inability of 2.5D Digital Elevation Models (DEMs) to model complex internal structures like caves or overhangs [S1]. Unlike its siblings, SBRT uses 1D-like structures (textures) which favor prefetching and stack-level caching, resulting in better performance than single-texture approaches [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D terrain dataset defined on a grid of width $W$ and height $H$. The data consists of volumetric layers or geological samples (boreholes) that can be interpolated into layers [S1].
*   **Output:** A GPU-resident memory layout consisting of two 2D textures: an interval texture and an index texture [S1].

**Data Structures:**
The grid dimensions are $W \times H$ (denoted as `gridancho` $\times$ `gridalto` in sources) [S1]. For each grid cell $(x,y)$, the structure stores a list of intervals. Each interval is defined by two values:
1.  **Start Height ($z$):** The z-coordinate where the interval begins.
2.  **Attribute:** A material or property ID [S1].

This is analogous to Run-Length Encoding (RLE) along the z-axis [S1]. The memory layout utilizes two 2D textures:
1.  **Interval Texture (`Textura de intervalos`):** Stores the interval data (start height and attribute) [S1].
2.  **Index Texture (`Textura de índices`):** Stores indices to spatially identify stack starts without fixed-size interval storage [S1].

**Implementation Approaches (Enfoques):**
The source describes five specific memory layout approaches ("Enfoques") for implementing SBRT on GPU hardware [S1]:

*   **Enfoque 1:** Uses two separate 2D textures for intervals and indices. The grid dimensions are $W \times H$. The last cells of the last row may remain unoccupied, wasting memory [S1].
*   **Enfoque 2:** Uses a single texture where regions of $(3 \times 2)$ texels are determined by the size of the largest stack (e.g., 5 intervals). The texture contains these fixed regions [S1].
*   **Enfoque 3 & 4:** Similar to Enfoque 2 but with different texel region configurations. Enfoque 3 has restrictions and cannot be used for especially large datasets due to maximum texture size limits of graphics cards [S1].
*   **Enfoque 5:** Uses two separate Shader Storage Buffer Objects (SSBOs) for intervals and indices, similar to Enfoque 1 but using SSBOs instead of textures [S1].

**Rendering Algorithm:**
The visualization method is based on GPU raycasting [S1]. Rays are launched from the virtual camera through each pixel of the output buffer, composing colors sampled along the ray [S1]. The structure is introduced in a compact and complete manner into GPU memory, avoiding data transfer between CPU and GPU [S1].

**Pseudocode:**
The following pseudocode outlines the data preparation for Enfoque 1, the primary approach described with specific memory layout details.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare_sbrt_terrain(grid_width, grid_height, terrain_data):
    """
    Prepares the SBRT data structure for Enfoque 1.
    
    Args:
        grid_width (int): W (gridancho)
        grid_height (int): H (gridalto)
        terrain_data (dict): Mapping of (x, y) -> list of intervals.
                             Each interval is (start_z, attribute_id).
                             
    Returns:
        interval_texture_data (list): Flattened list of interval data.
        index_texture_data (list): Flattened list of index data.
    """
    # Initialize textures with zeros or nulls
    # Size is grid_width * grid_height for each texture
    interval_texture_data = [None] * (grid_width * grid_height)
    index_texture_data = [None] * (grid_width * grid_height)
    
    # Iterate over every cell in the grid
    for y in range(grid_height):
        for x in range(grid_width):
            # Get intervals for this (x,y) stack
            # If no data exists for this cell, it remains empty/null
            intervals = terrain_data.get((x, y), [])
            
            if not intervals:
                continue
                
            # Store the intervals in the interval texture
            # The source implies a direct mapping or simple packing
            # We store the list of intervals. 
            # Note: The source mentions "Textura de intervalos" and "Textura de índices"
            # but does not give a specific formula for how the index texture 
            # points to the interval texture for variable-length stacks in Enfoque 1.
            # It states: "utilizando spatial indexing to identify stack starts 
            # without fixed-size interval storage."
            
            # For Enfoque 1, we assume the index texture stores a pointer/offset
            # or the interval texture stores the data sequentially per stack.
            # Given the lack of specific packing formula in sources, we note the structure.
            
            # [G] Implementation detail: Typically, index texture would store 
            # the starting offset in the interval texture for this stack.
            # Here we store the intervals directly if the layout is dense, 
            # or use the index texture to locate them.
            
            # Based on "Enfoque 1" description:
            # It uses two 2D textures. 
            # We will store the interval data in the interval texture.
            # We will store an index in the index texture.
            
            # Since the exact packing formula for variable length stacks in 
            # Enfoque 1 is not explicitly defined beyond "spatial indexing",
            # we represent the logical storage.
            
            interval_texture_data[y * grid_width + x] = intervals
            
            # [G] The index texture likely stores the count or start offset.
            # Without explicit formula, we store the number of intervals as a simple index.
            index_texture_data[y * grid_width + x] = len(intervals)

    return interval_texture_data, index_texture_data

def render_sbrt_raycast(ray_origin, ray_direction, interval_texture, index_texture, grid_width, grid_height):
    """
    Renders a pixel using SBRT raycasting.
    
    Args:
        ray_origin: 3D vector
        ray_direction: 3D vector
        interval_texture: 2D array of interval lists
        index_texture: 2D array of indices/counts
        grid_width: int
        grid_height: int
        
    Returns:
        color: 3D vector (RGB)
    """
    # 1. Intersect ray with the grid bounds to find entry/exit points
    # [G] Standard ray-box intersection logic required here.
    # Let t_min and t_max be the entry and exit distances along the ray.
    
    # 2. Determine the grid cells (x,y) the ray passes through
    # [G] Ray marching or grid traversal algorithm (e.g., Amanatides & Woo)
    # is needed to iterate through the (x,y) cells along the ray path.
    
    # 3. For each (x,y) cell intersected:
    #    a. Retrieve intervals from interval_texture at (x,y)
    #    b. Retrieve index/count from index_texture at (x,y)
    #    c. For each interval (start_z, attr):
    #       i. Calculate the z-range of the interval.
    #       ii. Intersect the ray segment with this z-range.
    #       iii. If intersection exists, sample the material property (attr).
    #       iv. Accumulate color using volume rendering equation (e.g., front-to-back compositing).
    
    # 4. Return accumulated color.
    pass
```

**Parameters and Edge Cases:**
*   **Grid Dimensions:** SBRT does not require power-of-two dimensions, unlike octrees which require padding (e.g., 200x250x320 becomes 512x512x512) [S1].
*   **Memory Waste:** In Enfoque 1, the last cells of the last row may remain unoccupied, wasting memory [S1].
*   **Material Mixing:** SBRT handles heterogeneous materials (mixing/overlapping) more simply at the stack level than hierarchical structures like octrees [S1].
*   **Large Datasets:** Enfoque 3 cannot be used for especially large datasets due to maximum texture size limits [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** SBRT uses significantly less memory than voxel models (e.g., 2.628 MB vs 61.035 MB for Terreno 1) and less than octrees (e.g., 2.628 MB vs 20.004 MB for Terreno 1) [S1]. It consumes less than 10% of the memory of an equivalent voxel model and between 13% and 30% of an octree [S1].
*   **Performance:** Enfoques 1 and 5 (using 1D-like structures/SSBOs) achieve the best performance (279-61 fps for Enfoque 5, 256-55 fps for Enfoque 1) due to favorable prefetching and stack-level caching [S1]. Enfoques 2, 3, and 4 (single-texture approaches) have worse performance due to lack of cache-friendly organization [S1].

**Strengths and Limitations:**
*   **Strengths:** Compact memory footprint; supports non-power-of-two dimensions; supports heterogeneous materials; suitable for real-time visualization and geological operations (slicing, layer attenuation) [S1].
*   **Limitations:** Not suitable for volumetric data not organized in layers (e.g., medical data) [S1]; Enfoque 3 has size limits; Enfoque 1 has some memory waste at grid boundaries [S1].

## Background
**Origin and History:**
SBRT was proposed in the paper "Direct Volume Rendering of Stack-Based Terrains" by A. Graciano, A.J. Rueda, and F.R. Feito, published in Eurographics Proceedings in 2017 [S1].

**Usage:**
It is used for the visualization of terrains and geological structures, adapting raycasting techniques for efficient visualization and operations without auxiliary structures [S1]. It is particularly suited for modeling subsurface data obtained from geological boreholes (catas) and interpolated layer models [S1].

## Sources
[S1] Direct Volume Rendering of Stack-Based Terrains, https://doi.org/10.2312/ceig.20171207, fetched 2026-10-26
