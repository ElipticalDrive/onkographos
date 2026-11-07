# Polygon-based Volumetric Rendering with Depth Peeling

Also called: 3D Texture with Polygons, Slice-based Volume Rendering

## Overview
This method renders solid 3D bodies from volumetric data by constructing a mesh of textured polygons (slices) and applying a 3D texture, using an undocumented rendering option to handle transparency sorting [S1, unofficial]. It mimics volumetric rendering using false coloring and alpha blending on polygons, whereas true volumetric rendering requires a transfer function converting density accumulation to color per pixel [S1, unofficial]. It is distinguished from standard texture slicing by its explicit construction of 3D polygon geometry with vertex texture coordinates and reliance on depth peeling for correct translucent ordering [S1, unofficial].

## How it works
**Inputs:** A stack of 2D volumetric slice images (e.g., 8-bit TIFFs) and rendering parameters.
**Outputs:** A rendered 3D visualization of the volume.

**Data Structures:**
1.  **Volumetric Texture Data (`data`):** A packed 3D array of shape $(N_z, N_y, N_x, 4)$, where $N_z$ is the number of slices, and $N_x, N_y$ are the image dimensions. Each element is an RGBA value in the "RGB" color space, where the fourth channel is opacity derived from binarized intensity [S1, unofficial].
2.  **Polygon Mesh:** A list of `Polygon` primitives. Each polygon is a quadrilateral defined by 4 vertices in 3D space. Each vertex has an associated `VertexTextureCoordinate` mapping to the 3D texture [S1, unofficial].

**Parameters:**
*   `top`: The z-value for the x-y cutaway plane. Typical value: $0.4$ (in normalized 0-1 coordinates) [S1, unofficial].
*   `right`: The x-value for the y-z cutaway plane. Typical value: $0.4$ [S1, unofficial].
*   `rpad`: Padding for the cutaway decoration. Typical value: $-0.2$ [S1, unofficial].
*   `step`: Slice spacing (distance between polygon layers). Smaller values increase fidelity but memory usage. Typical value: $0.01$ [S1, unofficial].
*   `DepthPeelingLayers`: Limit on rendering translucent polygons. Typical value: $100$ [S1, unofficial].

**Steps:**
1.  **Import and Convert:** Import the slice images and convert them into a single packed array. Reverse the list orientation if necessary to match the coordinate system [S1, unofficial].
2.  **Colorize and Alpha:** For each slice:
    a. Apply a color function (e.g., "SunsetColors") to generate RGB values.
    b. Binarize the original intensity to generate an alpha channel.
    c. Combine RGB and Alpha into an "RGB" color space array [S1, unofficial].
3.  **Pack Texture:** Extract image data from the colored slices, reversing data orientation if required, and pack into a single 3D array `data` [S1, unofficial].
4.  **Generate Bottom Polygons:** Create polygons for the z-range $[0, \text{top}]$. For each $z$ from $0$ to $\text{top}$ with step `step`:
    *   Define vertices: $P_1=(0,0,z)$, $P_2=(1,0,z)$, $P_3=(1,1,z)$, $P_4=(0,1,z)$.
    *   Assign `VertexTextureCoordinates` identical to the vertex positions [S1, unofficial].
5.  **Generate Top Polygons:** Create polygons for the cutaway geometry. For each $x$ from $0$ to $\text{right}$ with step `step`:
    *   Define vertices: $P_1=(x,0,\text{top})$, $P_2=(x,1,\text{top})$, $P_3=(x,1,1)$, $P_4=(x,0,1)$.
    *   Assign `VertexTextureCoordinates` identical to the vertex positions [S1, unofficial].
6.  **Render:** Draw the polygons with the 3D texture applied. Set global opacity (e.g., $0.4$) and enable depth peeling with a layer limit (e.g., $100$) to ensure correct translucent sorting [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_with_depth_peeling(slices, top=0.4, right=0.4, rpad=-0.2, step=0.01, depth_peeling_layers=100):
    """
    Render volumetric data using polygon-based texture mapping and depth peeling.
    
    Args:
        slices: List of 2D numpy arrays (images) representing volume slices.
        top: Z-coordinate for the horizontal cutaway.
        right: X-coordinate for the vertical cutaway.
        rpad: Padding for the cutaway edge decoration.
        step: Spacing between polygon slices.
        depth_peeling_layers: Number of depth peeling passes for transparency.
        
    Returns:
        A list of rendering commands/primitives (conceptual).
    """
    # Step 1 & 2: Colorize and Alpha
    # Assuming slices are normalized 0-1 intensity images
    colored_slices = []
    for img in slices:
        # Apply color function (e.g., SunsetColors) to get RGB
        rgb = apply_color_function(img, "SunsetColors")
        # Binarize to get alpha (1 where intensity > threshold, else 0)
        alpha = binarize(img)
        # Combine into RGBA in RGB space
        rgba = combine_rgb_alpha(rgb, alpha, space="RGB")
        colored_slices.append(rgba)
    
    # Reverse to match orientation
    colored_slices.reverse()
    
    # Step 3: Pack texture data
    # Extract image data, reverse orientation if needed, and pack
    data = pack_array([extract_image_data(img, reverse=True) for img in colored_slices])
    
    # Step 4: Generate Bottom Polygons (z from 0 to top)
    bottom_polygons = []
    z = 0
    while z <= top:
        # Vertices in 3D space
        pts = [
            [0, 0, z],
            [1, 0, z],
            [1, 1, z],
            [0, 1, z]
        ]
        # VertexTextureCoordinates match the polygon points
        poly = Polygon(vertices=pts, vertex_texture_coordinates=pts)
        bottom_polygons.append(poly)
        z += step
        
    # Step 5: Generate Top Polygons (x from 0 to right)
    top_polygons = []
    x = 0
    while x <= right:
        # Vertices defining the cutaway geometry
        pts = [
            [x, 0, top],
            [x, 1, top],
            [x, 1, 1],
            [x, 0, 1]
        ]
        poly = Polygon(vertices=pts, vertex_texture_coordinates=pts)
        top_polygons.append(poly)
        x += step
        
    # Step 6: Render Setup
    # Define cutaway decoration polygon (optional visual aid)
    cutaway_pts = [
        [right, 0, top],
        [1 + rpad, 0, top],
        [1 + rpad, 1, top],
        [right, 1, top],
        [right, 1, 1],
        [right, 0, 1]
    ]
    cutaway_poly = Polygon(vertices=cutaway_pts, edge_form=EdgeForm(opacity=0.6, color=RGB(1, 0.8, 0.1), thickness=2), face_form=FaceForm(opacity=0))
    
    # Assemble rendering commands
    rendering_commands = [
        SetOpacity(0.4),
        SetTexture(data),
        SetEdgeForm(None),
        *bottom_polygons,
        *top_polygons,
        cutaway_poly,
        SetPlotRange([[0, 1 + rpad], [0, 1], [0, 1]]),
        SetLighting("Neutral"),
        SetBackground(RGB(0, 0, 0)),
        SetRotationAction("Clip"),
        SetSphericalRegion(True),
        SetBoxStyle(Directive(opacity=0.2, color=RGB(1, 1, 1))),
        SetImageSize([400, 400]),
        SetBaseStyle(RenderingOptions={"DepthPeelingLayers": depth_peeling_layers}),
        SetBoxRatios([1 + rpad, 1, 0.9])
    ]
    
    return rendering_commands

def apply_color_function(image, name):
    # Placeholder for color mapping logic
    # Returns RGB array of same shape as image
    pass

def binarize(image):
    # Returns binary mask (0 or 1)
    pass

def combine_rgb_alpha(rgb, alpha, space):
    # Combines RGB and Alpha into a single array
    # Ensures Alpha is treated as opacity
    pass

def pack_array(list_of_arrays):
    # Converts list of arrays into a single packed 3D array
    pass

def extract_image_data(img, reverse=False):
    # Extracts raw image data, optionally reversing orientation
    pass

class Polygon:
    def __init__(self, vertices, vertex_texture_coordinates=None, edge_form=None, face_form=None):
        self.vertices = vertices
        self.vertex_texture_coordinates = vertex_texture_coordinates
        self.edge_form = edge_form
        self.face_form = face_form

class EdgeForm:
    def __init__(self, opacity, color, thickness):
        self.opacity = opacity
        self.color = color
        self.thickness = thickness

class FaceForm:
    def __init__(self, opacity):
        self.opacity = opacity

class RGB:
    def __init__(self, r, g, b):
        self.r = r
        self.g = g
        self.b = b

def SetOpacity(val):
    pass

def SetTexture(data):
    pass

def SetEdgeForm(form):
    pass

def SetPlotRange(range_3d):
    pass

def SetLighting(mode):
    pass

def SetBackground(color):
    pass

def SetRotationAction(action):
    pass

def SetSphericalRegion(flag):
    pass

def SetBoxStyle(style):
    pass

def SetImageSize(size):
    pass

def SetBaseStyle(style):
    pass

def SetBoxRatios(ratios):
    pass
```

## Operating characteristics
*   **Time Complexity:** Rendering time scales with the number of polygons ($N_{poly} \approx \frac{\text{top}}{\text{step}} + \frac{\text{right}}{\text{step}}$) and the depth peeling layers ($L$). The complexity is roughly $O(N_{poly} \cdot L)$ [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the 3D texture data ($N_z \cdot N_y \cdot N_x \cdot 4$ bytes) and the polygon mesh. For a typical dataset (e.g., CThead, 8-bit TIFFs), ~500MB of memory is required [S1, unofficial].
*   **Measured Performance:** Performance is limited by the Front End's ability to handle large polygon counts and depth peeling passes. Decreasing `step` or increasing `DepthPeelingLayers` and `ImageSize` can crash the Front End due to memory/resource limits [S1, unofficial].
*   **Strengths:** Simple to implement using standard graphics primitives; leverages existing 3D texture hardware acceleration; provides a visual approximation of volumetric rendering without complex ray-casting.
*   **Limitations:** Not true volumetric rendering (no transfer function-based density accumulation); quality depends heavily on slice spacing (`step`) and depth peeling layers; prone to crashing with high resolution or fine step sizes; limited flexibility compared to dedicated volume rendering engines [S1, unofficial].
*   **When to Use:** Quick visualization of volumetric data when true volume rendering is unavailable or too complex; educational purposes to demonstrate texture mapping and transparency.
*   **When Not to Use:** When accurate density-based visualization is required; when high resolution and fine detail are needed without performance degradation; when working with very large datasets that exceed available memory.

## Background
This technique was presented as a solution for rendering solid 3D bodies using volumetric rendering in Mathematica, leveraging its 3D graphics capabilities and undocumented rendering options [S1, unofficial]. It was developed in response to the limited native volumetric rendering support in earlier versions of Mathematica (e.g., version 8), which lacked proper transfer functions and density accumulation [S1, unofficial]. The method mimics volumetric rendering by using false coloring and alpha blending on polygons, serving as a workaround until more advanced features like `CUDAVolumetricRender` became available [S1, unofficial]. It relies on the `DepthPeelingLayers` rendering option, which is undocumented and controls the limit on rendering translucent polygons [S1, unofficial].

## Sources
[S1] How to show solid bodies using volumetric rendering?, https://mathematica.stackexchange.com/questions/6247, fetched 2026-10-30 (unofficial)
