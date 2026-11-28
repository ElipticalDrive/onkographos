# Polygon-based Volumetric Rendering with Depth Peeling

Also called: 3D Texture with Polygons, Slice-based Volume Rendering

## Overview
Polygon-based volumetric rendering with depth peeling is a visualization method for scalar fields that constructs semi-transparent polygon slices draped with 3D textures and piles them from back to front, avoiding the heavy numerical integration of per-pixel ray casting [S1]. It mimics true volumetric rendering by using false coloring and alpha blending on polygons, relying on an undocumented rendering option to handle transparency sorting [S2, unofficial]. It differs from [3D Texture Slicing with Back-to-Front Blending](12611-3d-texture-slicing-with-back-to-front-blending.md) by explicitly constructing geometric polygon meshes rather than relying on hardware texture slicing, and from [Object Aligned Planes Volume Rendering](11411-object-aligned-planes-volume-rendering.md) by using a fixed grid-aligned slice structure rather than object-aligned planes [S2, unofficial].

## How it works
**Inputs:** A stack of volumetric slice images (e.g., 8-bit TIFFs) representing a scalar field, and rendering parameters including `top` (z-value for x-y cutaway), `right` (x value for y-z cutaway), `rpad` (padding), `step` (slice spacing), and `DepthPeelingLayers` (limit on translucent polygon rendering) [S2, unofficial].

**Outputs:** A rendered image of the semi-transparent 3D body [S2, unofficial].

**Data Structures:**
1.  **Volumetric Texture Data (`data`):** A packed 3D array of RGB values. Each slice is colorized and combined with an alpha channel derived from binarized intensity. The data orientation is reversed to match the coordinate system [S2, unofficial].
2.  **Polygon Mesh:** A list of 4-point polygons (quads) defined by vertex coordinates and `VertexTextureCoordinates` that map the 3D texture [S2, unofficial].

**Steps:**
1.  **Import and Preprocess Data:** Import volumetric slice data (e.g., 8-bit TIFFs) and convert to a packed array. Apply colorization using a specific `ColorFunction` and generate an alpha channel using `Binarize`. Combine into "RGB" space and reverse the list to match orientation [S2, unofficial].
2.  **Generate Bottom Part Polygons:** Create a table of 4-point polygons for $z$ from $0$ to `top` with `step` spacing. The vertices are defined as:
    $$ P_{bottom}(z) = \{ (0, 0, z), (1, 0, z), (1, 1, z), (0, 1, z) \} $$
    Assign `VertexTextureCoordinates` matching these points [S2, unofficial].
3.  **Generate Top Part Polygons:** Create a table of 4-point polygons for $x$ from $0$ to `right` with `step` spacing. The vertices define the cutaway geometry:
    $$ P_{top}(x) = \{ (x, 0, top), (x, 1, top), (x, 1, 1), (x, 0, 1) \} $$
    Assign `VertexTextureCoordinates` matching these points [S2, unofficial].
4.  **Configure Rendering:** Set global graphics options including `Opacity[.4]` for overall transparency, `Texture[data]` for the volumetric texture, and `BaseStyle -> {RenderingOptions -> {"DepthPeelingLayers" -> n}}` where $n$ is typically 100 [S2, unofficial].
5.  **Render:** Render the piled slices. The depth peeling algorithm handles the sorting of translucent polygons to ensure correct blending [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Import and Preprocess
# slices: list of 2D arrays (images)
# color_func: function mapping intensity to RGB
# binarize_threshold: float for alpha generation

colored_slices = []
for img in slices:
    # Colorize the image
    rgb = color_func(img)
    # Generate alpha from binarized image
    alpha = binarize(img)
    # Combine into RGB with alpha (assuming 4-channel or separate alpha handling)
    # Note: Source specifies "RGB" space for ColorCombine, implying alpha is handled separately or as 4th channel
    combined = combine_rgb_alpha(rgb, alpha)
    colored_slices.append(combined)

# Reverse to match orientation
colored_slices.reverse()

# Extract image data and pack into a single 3D array
# DataReversed ensures correct orientation mapping
data = pack_array([extract_image_data(img, data_reversed=True) for img in colored_slices])

# Step 2 & 3: Generate Polygons
# Parameters
top = 0.4
right = 0.4
rpad = -0.2
step = 0.01

bottom_polygons = []
for z in range(0, top, step):
    # Vertices: (0,0,z), (1,0,z), (1,1,z), (0,1,z)
    pts = [(0, 0, z), (1, 0, z), (1, 1, z), (0, 1, z)]
    # VertexTextureCoordinates match the polygon points
    bottom_polygons.append(Polygon(pts, vertex_texture_coords=pts))

top_polygons = []
for x in range(0, right, step):
    # Vertices: (x,0,top), (x,1,top), (x,1,1), (x,0,1)
    pts = [(x, 0, top), (x, 1, top), (x, 1, 1), (x, 0, 1)]
    top_polygons.append(Polygon(pts, vertex_texture_coords=pts))

all_polygons = bottom_polygons + top_polygons

# Step 4: Configure Rendering
render_params = {
    "opacity": 0.4,
    "texture": data,
    "depth_peeling_layers": 100,
    "lighting": "Neutral",
    "background": "Black"
}

# Step 5: Render
# The renderer uses depth peeling to sort and blend the translucent polygons
render(all_polygons, render_params)
```

**Parameters:**
*   `DepthPeelingLayers`: Undocumented option controlling the limit on rendering translucent polygons. Typical value: 100 [S2, unofficial].
*   `step`: Slice spacing. Smaller values increase detail but risk crashing the frontend [S2, unofficial].
*   `top`: Z-value for x-y cutaway [S2, unofficial].
*   `right`: X value for y-z cutaway [S2, unofficial].

**Pitfalls:**
*   Decreasing `step` size or increasing `DepthPeelingLayers` and `ImageSize` can crash the Front End due to memory/resource limits [S2, unofficial].
*   This method mimics volumetric rendering using false coloring and alpha blending; it is not true volumetric rendering which requires a transfer function converting density accumulation to color per pixel [S2, unofficial].

## Subtopics

### Shell Rendering with Levels of Detail
A method for rendering semi-transparent details on animated surfaces by using levels of detail (mips) for volumetric textures, handling semi-transparent voxels that appear on edges during mip downscaling via order independent transparency [S3]. It discards invisible parts using hardware z-test to reduce rendering time, splits each depth peeling pass into multiple steps to minimize expensive lighting calculations, and finds the first $K$ semi-transparent layers (empirically 3-4) for stable results [S3]. It adds an additional resolving pass to fully utilize hardware z-buffering and reduce overdraw [S3]. Unlike methods requiring tetrahedron sorting, it uses depth peeling with z-buffering to avoid sorting [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, it avoids the per-pixel ray casting integration of classical volume rendering, offering higher speed [S1].
*   **Memory Complexity:** Requires significant memory for the volumetric texture data and polygon mesh. Decreasing step size increases memory usage and can crash the frontend [S2, unofficial].
*   **Performance:** High-speed compared to ray casting [S1]. Performance is limited by the number of depth peeling layers and the number of polygons [S2, unofficial].
*   **Strengths:** High-speed rendering, interactive transfer function editing via 3D GUI [S1], suitable for VR environments like CAVE [S1].
*   **Limitations:** Not true volumetric rendering (mimics effect) [S2, unofficial], undocumented rendering options, risk of crashing with high resolution [S2, unofficial].
*   **When to use:** When high-speed visualization of scalar fields is needed in VR or interactive settings, and true volumetric accuracy is not critical [S1][S2, unofficial].
*   **When not to use:** When true density accumulation and transfer function-based color mapping per pixel is required [S2, unofficial].

## Background
The method was developed for the VFIVE system, originally designed for stationary field analysis but extended to support animation [S1]. It uses three-dimensional texture mapping to realize high-speed volume rendering, piling polygon slices from back to front [S1]. The implementation includes an interactive 3-D GUI for editing transfer functions in the CAVE room [S1]. The specific implementation details using Mathematica's undocumented `DepthPeelingLayers` option were shared in an unofficial community answer [S2, unofficial].

## Sources
[S1] IMMERSIVE VR VISUALIZATIONS BY VFIVE PART 1: DEVELOPMENT, https://doi.org/10.1142/s1793962313400035, fetched 2026-10-29
[S2] How to show solid bodies using volumetric rendering?, https://mathematica.stackexchange.com/questions/6247, fetched 2026-10-30 (unofficial)
[S3] Real-time Rendering of Small-scale Volumetric Structure on Animated Surfaces, https://doi.org/10.20948/graphicon-2021-3027-64-73, fetched 2026-10-26
