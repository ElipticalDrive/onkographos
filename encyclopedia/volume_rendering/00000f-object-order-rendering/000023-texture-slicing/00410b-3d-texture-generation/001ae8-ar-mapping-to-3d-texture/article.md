# AR mapping to 3D texture

Also called: 3D texture mapping, volume texture mapping

## Overview
AR mapping to 3D texture is a volume rendering technique that stores pre-computed color and opacity values directly into a 3D texture, allowing the graphics hardware to perform the volume integration via linear alpha blending during rasterization [S1, unofficial]. It solves the problem of non-linear volume composition formulas by replacing them with hardware-accelerated texture lookups and blending operations [S1, unofficial]. Unlike standard ray casting which computes integration per-voxel along rays, this method renders the volume as a set of parallel polygons (slices) mapped against the 3D texture, trading bit-accuracy for high performance [S2].

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar volume dataset (voxels).
- Output: A 2D image frame buffer.

**Data Structures:**
- **3D Texture:** A 3D texture object where each texel stores the pre-classified color ($C$) and opacity ($\alpha$) of the corresponding voxel. The texture coordinates map linearly to the voxel grid indices.
- **Polygon Set:** A collection of parallel polygons (typically quads) representing the slices of the volume. The number of polygons corresponds to the depth of the volume along the view direction.

**Steps:**
1. **Pre-classification:** For each voxel in the 3D volume, apply the transfer function to compute the color $C$ and opacity $\alpha$. Store these values in the 3D texture.
2. **Coordinate Definition:** Define texture coordinates for the polygon set once. The mapping is linear, avoiding the need for non-linear formulae that would require recalculation at each rotation [S1, unofficial].
3. **Rotation:** Rotate the set of parallel polygons to match the desired view direction. Because the texture coordinates are defined once and the mapping is linear, they do not need to be recalculated for the rotation [S1, unofficial].
4. **Display List:** Place the set of polygons and the texture binding commands into a display list (a group of precomputed OpenGL commands cached for later execution) to improve performance when redrawing the geometry multiple times [S1, unofficial].
5. **Rendering:** Render the polygons. The graphics hardware performs linear alpha blending as it composites the fragments from the front to the back, effectively integrating the volume [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the rendering loop
def render_volume_3d_texture(volume_data, transfer_function, view_matrix):
    # 1. Pre-classify volume into 3D texture
    # This step is typically done once or when data changes
    texture = create_3d_texture(volume_data.shape)
    for x, y, z in volume_data.indices():
        color, alpha = transfer_function(volume_data[x, y, z])
        texture.set_texel(x, y, z, color, alpha)
    
    # 2. Define polygon set (slices)
    # Number of slices equals the depth of the volume along the view direction
    num_slices = volume_data.shape[depth_axis]
    polygons = []
    for i in range(num_slices):
        # Create a quad representing the slice at index i
        # Texture coordinates are linearly mapped to [0, 1]
        poly = create_quad_slice(i, volume_data.shape)
        polygons.append(poly)
    
    # 3. Create display list for performance
    display_list = create_display_list()
    display_list.bind_texture(texture)
    for poly in polygons:
        display_list.draw(poly)
    
    # 4. Render loop
    while not done:
        # Rotate the view matrix
        view_matrix = update_view_matrix(view_matrix)
        
        # Bind the display list and render
        # The hardware handles alpha blending and texture lookup
        gl_use_display_list(display_list)
        gl_set_view_matrix(view_matrix)
        gl_render()
```

**Parameters:**
- **Framebuffer Resolution:** A color/RGBA resolution of at least 16 bits or more is desirable for good quality [S2].
- **Texture Coordinates:** Linear mapping is used. No specific parameters other than the volume dimensions are needed for the mapping itself [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The quality of the display is limited due to the lack of bit accuracy inherent in hardware blending [S2].
- **Off-Axis Views:** Image quality degrades at off-axis views due to the nature of the parallel projection and interpolation [S2].
- **Interpolation Cost:** At higher magnifications, the cost of trilinear interpolations may overtake footprint mappings, leading to a critical screen size where performance drops [S2].
- **Estimation Error:** The estimation method (linear mapping) is likely worse than Wilson's depth integration, but visualizations remain effective [S1, unofficial].

## Subtopics

### [Gradient-Isosurface Texture Packing](889-gradient-isosurface-texture-packing.md)
Gradient-Isosurface Texture Packing is a data layout strategy for 3D volume textures that stores pre-computed gradient vectors in the RGB components and the original scalar isovalue in the Alpha component. This allows a single texture to provide both the data for transfer function indexing and the normals for shading, avoiding the need for a second texture copy or expensive runtime gradient calculation. It differs from [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md) by embedding shading data directly into the volume representation, and from [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md) by relying on hardware interpolation of pre-computed gradients rather than software-based ray casting optimizations.

### [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md)
Röttger et al. Rendering is a technique that mitigates the high computational cost of volume integration by approximating the rendering integral and moving the computation to a pre-process. It computes a 3D lookup table of colors and opacities parameterized by front scalar value, back scalar value, and distance, which is stored in a 3D texture for hardware-accelerated lookups during rendering. This method builds on [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md) by using the 3D texture not just for static data storage, but to evaluate the volume composition operator directly via texture interpolation. It is distinguished from standard ray casting by replacing per-voxel integration with texture lookups, and from pure pre-classification by supporting arbitrary transfer functions including isosurface effects through the parameterization of the lookup table.

### Uniform Grid
The Uniform Grid variant uses a structured (X, Y, Z) float32 tensor to store scalar values, with an associated bounding box defined by `aabb_min` (default [-1, -1, -1]) and `aabb_max` (default [1, 1, 1]) [S3, unofficial]. It provides methods `get_points` returning a (X, Y, Z, 3) float32 tensor of 3D coordinates and `get_values` returning the scalar tensor [S3, unofficial]. The `set_values` method accepts a (X, Y, Z) float32 tensor to update scalar values, with a `default_value` (default float max) for initialization [S3, unofficial]. This structure is often used as the underlying data representation for the volume data fed into the 3D texture mapping process.

## Operating characteristics
- **Time Complexity:** Rendering time is sub-second for moderately-sized datasets, making it a high-performance renderer [S2]. The cost is largely independent of the number of rays, relying instead on the number of polygons and texture lookups [S2].
- **Memory Complexity:** Requires storing a 3D texture for the volume data. The memory usage is proportional to the volume resolution.
- **Measured Performance:** Achieves sub-second rendering times for moderately-sized datasets [S2]. However, quality is limited due to lack of bit accuracy [S2].
- **Scaling:** Performance degrades at higher magnifications due to increased trilinear interpolation costs [S2]. Off-axis views also suffer from quality degradation [S2].
- **Strengths:** High performance, simple implementation using standard graphics hardware, effective visualizations [S1, unofficial].
- **Limitations:** Limited quality due to lack of bit accuracy, degradation at off-axis views, and potential performance issues at high magnifications [S2].

## Background
AR mapping to 3D texture is a technique described in the context of volume visualization via variable-detail non-photorealistic illustration [S1, unofficial]. It leverages the capabilities of 3D texture mapping in graphics hardware to perform volume integration. The method is discussed in practical evaluations of volume rendering algorithms, where it is compared with shear-warp and raycasting [S2].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S3] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
