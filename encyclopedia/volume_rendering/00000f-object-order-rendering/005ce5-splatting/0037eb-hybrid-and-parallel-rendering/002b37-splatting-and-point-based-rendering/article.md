# Splatting and Point-based Rendering

Also called: Accelerating Splatting Algorithm Based on Multi-Texture Mapping, Multi-Texture Volume Rendering, Texture Splatting with Hardware Blending

## Overview
Splatting and Point-based Rendering is an object-order volume rendering method where 3D data volumes are projected onto the 2D screen or an intermediate texture plane and blended to form the final image [S1, unofficial][S2]. It solves the problem of efficiently rendering large volumetric datasets by leveraging specialized hardware for compositing, avoiding the need for explicit intermediate geometry or thresholding [S1, unofficial][S2]. Unlike ray-order methods that cast rays through the volume, this method projects primitives (voxels or points) directly, allowing for fast hardware-accelerated alpha blending [S1, unofficial][S2]. It sets itself apart from its siblings by serving as the foundational object-order technique that other methods, such as [Ray-Driven Splatting](1156-ray-driven-splatting.md) or [Hybrid Volume Rendering with Hierarchical Representation](12636-hybrid-volume-rendering-with-hierarchical-representation.md), build upon or optimize [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D data volume (scalar field) defined in object coordinates, along with a transfer function mapping sample values to color and opacity [S1, unofficial][S2].
*   **Output:** A 2D rendered image on the screen [S1, unofficial][S2].

**Data Structures:**
*   **Volume Data:** A 3D array of scalar values.
*   **Footprint Lookup Table:** A table storing pre-computed filter weights for the splat footprint [S2].
*   **Texture Slices:** 2D texture maps representing segments of the volume, stored in RGBA mode to include the opacity channel [S2].

**Numbered Steps:**

1.  **Transformation:** Transform the data volume from object coordinates to world coordinates [S2].
2.  **Color and Opacity Mapping:** Map sample values to color and opacity based on density using a transfer function [S1, unofficial][S2].
3.  **Footprint Computation:** Compute filter weights for the splat footprint and store them in a lookup table [S2]. The filter is typically Gaussian and proportional to the distance from the projection plane [S1, unofficial].
4.  **Initialization:** Initialize texture points to RGBA mode and set the original alpha value of every point to zero [S2].
5.  **Data Volume Division:** Divide the data volume into segments or regions [S2].
6.  **Processing Segments:** For each segmented data set in the volume data:
    a.  **Depth Sort:** Sort the voxels in front-to-back order (z-value from large to small) [S2].
    b.  **Layer Grouping:** Group points with the same z-coordinate into layers [S2].
    c.  **Projection:** For each layer:
        i.  Calculate the contribution of each voxel using the footprint weights [S2].
        ii. Project the voxels to the texture plane [S2]. The texture plane is oriented parallel to the screen plane to avoid popping and splatting overlay artifacts [S2].
    d.  **Texture Slice Creation:** After all layers in one region are projected to the texture plane, create one texture slice representing that region [S2].
7.  **Blending:** For each texture slice:
    a.  Blend the texture slices in front-to-back order [S2].
    b.  Turn on alpha blending during this step [S2].

**Formulas:**
*   **Filter Weights (Footprint):** Formula (6) is used to compute filter weights for the footprint lookup table [S2]. The specific mathematical expression is not provided in the sources, but it is described as a Gaussian filter proportional to the distance from the plane [S1, unofficial].
*   **Data Volume Division:** Formula (7) is used to divide the data volume [S2]. The specific expression is not provided.
*   **Projection to Texture Plane:** Formula (12) is used to project points to the texture plane [S2]. The specific expression is not provided.

**Parameters:**
*   **Texture Mode:** RGBA, required to include the opacity channel of every data sample for translucent image generation [S2].
*   **Texture Plane Orientation:** Parallel with the screen plane to avoid popping and splatting overlay [S2].
*   **Polygon Vertex Constraint:** x and y coordinates are equal for all vertices, except z, ensuring texture slices are paralleled with the screen plane [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatting_rendering(volume_data, transfer_function, projection_matrix):
    # 1. Transformation: Transform data volume from object coordinates to world coordinates
    world_coords = [transform_point(v, projection_matrix) for v in volume_data]
    
    # 2. Color and Opacity Mapping: Map sample values to color and opacity based on density
    colored_volume = [transfer_function(v.density) for v in world_coords]
    
    # 3. Footprint Computation: Compute filter weights for the footprint lookup table
    # Formula (6) computes Gaussian weights proportional to distance from plane. Not given in sources.
    footprint_table = compute_footprint_weights()
    
    # 4. Initialization: Initialize texture points to RGBA mode, alpha set to zero
    texture_slices = []
    
    # 5. Data Volume Division: Divide the data volume using Formula (7). Not given in sources.
    segmented_data = divide_volume(colored_volume)
    
    for segment in segmented_data:
        # 6a. Depth Sort: Sort voxels in front-to-back order (z large to small)
        sorted_segment = sorted(segment, key=lambda v: v.z, reverse=True)
        
        # 6b. Layer Grouping: Group points with the same z into layers
        layers = group_by_z(sorted_segment)
        
        # 6c. Projection
        texture_buffer = initialize_texture_buffer_rgba()
        for layer in layers:
            for voxel in layer:
                # 6c-i. Calculate contribution with footprint
                contribution = apply_footprint(voxel, footprint_table)
                
                # 6c-ii. Project to texture plane using Formula (12). Not given in sources.
                project_to_texture(voxel, contribution, texture_buffer)
        
        # 6d. Make texture slice
        texture_slices.append(texture_buffer)
        
    # 7. Blending: Blend texture slices in front-to-back order with alpha blending
    final_image = initialize_screen_buffer()
    for slice in texture_slices:
        final_image = blend_textures(final_image, slice)
        
    return final_image

# Helper definitions
def transform_point(v, matrix):
    # Standard 4x4 matrix multiplication for object-to-world transformation [G]
    return [sum(v[i] * matrix[i][j] for i in range(4)) for j in range(4)]

def compute_footprint_weights():
    # Computes Gaussian filter weights proportional to distance from projection plane
    # Formula (6) is referenced but not provided in sources.
    return {}

def divide_volume(data):
    # Divides volume data into segments/regions
    # Formula (7) is referenced but not provided in sources.
    return [data]

def group_by_z(sorted_data):
    # Groups voxels sharing the same z-coordinate into layers
    layers = []
    current_z = None
    current_layer = []
    for v in sorted_data:
        if v.z != current_z:
            if current_layer:
                layers.append(current_layer)
            current_z = v.z
            current_layer = [v]
        else:
            current_layer.append(v)
    if current_layer:
        layers.append(current_layer)
    return layers

def initialize_texture_buffer_rgba():
    # Initializes a 2D buffer with RGBA mode, setting all alpha values to zero
    return [[(0.0, 0.0, 0.0, 0.0) for _ in range(BUFFER_W)] for _ in range(BUFFER_H)]

def apply_footprint(voxel, table):
    # Applies footprint filter weights to the voxel's contribution
    return voxel.density * table.get(voxel.distance, 1.0)

def project_to_texture(voxel, contribution, buffer):
    # Projects voxel to texture plane coordinates and accumulates contribution
    tx, ty = voxel.screen_x, voxel.screen_y
    buffer[ty][tx] = accumulate_rgba(buffer[ty][tx], contribution)

def initialize_screen_buffer():
    # Initializes the final screen buffer
    return [[(0.0, 0.0, 0.0, 0.0) for _ in range(BUFFER_W)] for _ in range(BUFFER_H)]

def blend_textures(final, slice):
    # Performs front-to-back alpha blending of a texture slice into the final image [G]
    for y in range(BUFFER_H):
        for x in range(BUFFER_W):
            src = slice[y][x]
            dst = final[y][x]
            alpha = src[3]
            final[y][x] = (
                dst[0] * (1 - alpha) + src[0] * alpha,
                dst[1] * (1 - alpha) + src[1] * alpha,
                dst[2] * (1 - alpha) + src[2] * alpha,
                dst[3] + src[3] * (1 - dst[3])
            )
    return final
```

**Edge Cases and Pitfalls:**
*   **Holes:** The discrete process of splatting can produce holes on the periphery or during extreme perspective projection [S1, unofficial]. This is countered by distributing the energy across multiple pixels via a footprint table, making adjustments before rendering [S1, unofficial].
*   **Popping and Overlay:** Popping and splatting overlay artifacts can occur if the texture plane is not parallel to the screen plane [S2].

## Subtopics

### [Ray-Box Intersection via Quad Rasterization](2253-ray-box-intersection-via-quad-rasterization.md)
A voxel rendering technique where each voxel is represented by a single point primitive sized to cover the voxel's projected area, with the actual voxel geometry (cube) intersected with a ray in the fragment shader using an optimized ray-box intersection test. It differs from sibling methods like [Sparse Voxel Octree Rendering via Square Splats](2993-sparse-voxel-octree-rendering-via-square-splats.md) by avoiding explicit cube face rasterization, instead relying on GPU point primitives and fragment-shader ray tracing to determine visibility.

### [Sparse Voxel Octree Rendering via Square Splats](2993-sparse-voxel-octree-rendering-via-square-splats.md)
Sparse Voxel Octree Rendering via Square Splats visualizes sparse voxel octrees (SVOs) by converting leaf voxels into square billboards (splats) using a geometry shader within a programmable OpenGL graphics pipeline. It differs from ray-tracing SVO renderers by relying on rasterization, and from point-sprite methods by explicitly generating square geometry to approximate voxel volume.

### [Voxel Point Sprite Rendering](4307-voxel-point-sprite-rendering.md)
Voxel Point Sprite Rendering is a technique that stores voxels as points (splatting) in a buffer and renders them using a geometry shader or point sprites, allowing for mipmapped voxel data to be drawn as points with a specific scale. It visualizes voxels as points with size and color based on solidity. The method reduces vertex traffic by using the `glPoint` primitive with the Point Sprites extension to create automatic quads, reducing vertex traffic from four to one.

### [Differentiable Point-Based Radiance Fields](7366-differentiable-point-based-radiance-fields.md)
Differentiable Point-Based Radiance Fields (DPBF) is a scene representation and rendering method that models radiance using a learned point cloud augmented with per-point spherical harmonic coefficients. It solves the problem of slow training and high memory usage in neural radiance fields by replacing volumetric ray-marching with a differentiable splat-based rasterizer, allowing for efficient view synthesis via a single forward pass. Unlike NeRF, which requires many evaluations per ray, DPBF uses forward mapping, resulting in training speeds up to 300x faster and memory usage under 10 MB for static scenes.

### [Adapted Splatting for Medical Applications](8546-adapted-splatting-for-medical-applications.md)
Adapted Splatting is a volume rendering method that optimizes standard splatting by culling voxels invisible to the observer and using a compressed data format to reduce memory usage for free voxels. It solves the high computational cost of processing all voxels in a model and the large memory footprint of storing sparse volumetric data. It sets itself apart from standard splatting by introducing a two-pass optimization: a depth-buffer-based occlusion culling pass that identifies only visible voxels, followed by rendering only those selected voxels, while simultaneously employing a run-length-like encoding for storage that groups consecutive free voxels. It builds on [Splatting and Point-based Rendering](4975-splatting-and-point-based-rendering.md) and differs from siblings such as [Ray-Box Intersection via Quad Rasterization](2253-ray-box-intersection-via-quad-rasterization.md), [Sparse Voxel Octree Rendering via Square Splats](2993-sparse-voxel-octree-rendering-via-square-splats.md), [Voxel Point Sprite Rendering](4307-voxel-point-sprite-rendering.md), and [Differentiable Point-Based Radiance Fields](7366-differentiable-point-based-radiance-fields.md) by relying on explicit voxel culling and a custom segment-based storage layout rather than hierarchical traversal or differentiable field representations.

### [Hybrid Volume Rendering with Hierarchical Representation](12636-hybrid-volume-rendering-with-hierarchical-representation.md)
A hardware-accelerated hybrid volume rendering scheme that provides a smooth transition between texture-based (raycasting) and point-based (splatting) techniques by dividing the volumetric dataset into hierarchical nodes, each rendered with the most efficient method based on a division criterion. It solves the problem of visual popping artifacts and performance inefficiencies inherent in using a single rendering mode for entire datasets by adapting the technique locally. Unlike pure splatting [Splatting and Point-based Rendering](4975-splatting-and-point-based-rendering.md) or pure raycasting, this method dynamically selects the approximation scheme for each subpart of the dataset to minimize visual variation while optimizing frame rate.

### Space-Filling Points
Using points to fill empty space in irregular grids, such as filling cells with Poisson distributed points [S3, unofficial]. The method involves filling cells of an irregular grid with Poisson distributed points and then performing projection via point-based rendering [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The method involves sorting voxels and projecting them to texture planes. The sources note that sorting segmented volume data is a significant part of the process [S2].
*   **Memory Complexity:** Requires storage for the volume data, footprint lookup table, and intermediate texture slices [S2].
*   **Measured Performance:** On a PC with a 1.8 GHz Pentium 4 CPU and 512M DRAM [S2]:
    *   Seismic data (128x128x64): Old method 9.672s, New method 1.917s, Accelerating ratio 5.045 [S2].
    *   CT Head data (256x256x255): Old method 3.198s, New method 0.518s [S2].
*   **Strengths:** Leverages specialized texture hardware for fast rendering and composing, avoiding the need to project voxels directly to the screen plane [S2]. It helps to zoom or un-zoom the image easily by controlling polygon size [S2].
*   **Limitations:** The discrete process can produce holes on the periphery or during extreme perspective projection [S1, unofficial]. Requires careful handling of texture plane orientation to avoid popping and splatting overlay [S2].
*   **When to use:** Suitable for large-scale volumetric data where hardware-accelerated blending is beneficial [S2].
*   **When not to use:** Not suitable when extreme perspective projection is required without additional footprint adjustments to prevent holes [S1, unofficial].

## Background
The method originates from the work of Crawfis and Max in 1993, who introduced "Texture Splats for 3D Scalar and Vector Field Visualization" [S2]. This work laid the foundation for using texture hardware to accelerate volume rendering by converting 3D data into 2D texture maps [S2]. The specific accelerating algorithm based on multi-texture mapping described in the notes builds upon this foundation, optimizing the projection and blending steps [S2].

## Sources
[S1] Direct Volume Rendering Direct Volume Rendering. What is volume rendering?. Accumulate information along 1 dimension line through volume. Volume rendering vs. isosurfaces. No intermediate geometry No , https://www.slideserve.com/lenci/direct-volume-rendering, fetched 2026-10-25 (unofficial)
[S2] An accelerating splatting algorithm based on multi-texture mapping for volume rendering, https://doi.org/10.1145/1174429.1174464, fetched 2026-11-05
[S3] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
