# Interpolation of Arbitrary Slices (Multi-Texture Blending)

Also called: Multi-planar reformatting (MPR) via multi-texturing, Viewport-aligned slice interpolation

## Overview
This method renders arbitrary slicing planes through a volume by interpolating between two adjacent object-aligned texture slices using multi-texturing hardware [S1]. It solves the problem of rendering non-axis-aligned cross-sections without requiring 3D texture storage or complex ray-casting, by decomposing the arbitrary slice into polygon strips that span only two 2D textures [S1]. It sets itself apart from single-texture approaches by reducing the polygon count by half, though it is constrained by memory bandwidth limitations [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Volumetric data stored in a sequence of 2D texture images (slices $i$ and $i+1$), a slicing plane defined in world space, and the bounding box of the volume [S1].
*   **Output:** A 2D image representing the cross-section of the volume along the arbitrary slicing plane [S1].

**Data Structures**
*   **Texture Images:** 2D textures containing the volumetric data for specific axial slices [S1].
*   **Polygon Strips:** Geometric primitives representing the intersection of the slicing plane with the volume, clipped to the boundaries of the two relevant texture slices [S1].
*   **Vertex Attributes:** Each vertex of a polygon strip carries:
    *   Position (projected to screen space) [G].
    *   Texture coordinates (mapped to the respective 2D texture) [G].
    *   Alpha value ($\alpha$): An interpolation factor where $\alpha=0$ corresponds to texture slice $i$ and $\alpha=1$ corresponds to texture slice $i+1$ [S1].

**Algorithm Steps**
1.  **Calculate Intersection Polygon:** Determine the cross-section of the arbitrary slicing plane with the volume's bounding box [S1].
2.  **Clip to Texture Boundaries:** Cut the intersection polygon into a set of polygon strips at the intersection lines with the object-aligned texture slices [S1].
3.  **Assign Vertex Attributes:** For each polygon strip, map the strip's vertices to the 2D textures of slice $i$ and slice $i+1$, and assign an alpha value to each vertex based on its position relative to the axial slices [S1]. An alpha value of 0 indicates the vertex is textured with slice $i$, and an alpha value of 1 indicates it is textured with slice $i+1$ [S1].
4.  **Interpolate Alpha (Gouraud Shading):** Use Gouraud shading to interpolate the alpha values across the polygon strip [S1]. The primary color alpha channel is interpolated linearly between the vertex alpha values [S1].
5.  **Multi-Texture Blending:** Use the register combiner hardware to blend the two texture images based on the interpolated alpha [S1].
    *   **General Combiner 0 Setup:** Input Register A is Texture 0 (Slice $i$), Input Register B is Primary Color Alpha (interpolated $\alpha$), Input Register C is Texture 1 (Slice $i+1$), and Input Register D is INVERT(B) (i.e., $1 - \alpha$) [S1]. The output is computed as $A \cdot B + C \cdot D$ [S1].
    *   This operation performs the linear interpolation: $\text{Result} = \text{Texture}_i \cdot (1-\alpha) + \text{Texture}_{i+1} \cdot \alpha$ [S1].
    *   **Final Combiner:** Blends the resulting RGB color with the secondary color (if used for shading) and sets the final alpha [S1]. Input Register A is Primary Color (RGB), Input Register B is Secondary Color, Input Register C is ZERO, and Input Register D is Constant Color 2 [S1]. The output is computed as $A \cdot B + C \cdot D$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_arbitrary_slice(slicing_plane, volume_bbox, textures):
    """
    Renders an arbitrary slice using multi-texture blending.
    textures: list of 2D texture objects, where textures[i] is slice i.
    """
    # Step 1: Calculate intersection polygon
    intersection_polygon = intersect_plane_bbox(slicing_plane, volume_bbox)
    
    # Step 2: Clip polygon into strips between adjacent slices
    slice_indices = find_intersecting_axial_slices(intersection_polygon, volume_bbox)
    
    for strip in clip_polygon_to_axial_strips(intersection_polygon, slice_indices):
        # Step 3: Assign vertex attributes
        for vertex in strip.vertices:
            slice_i = strip.lower_slice_index
            slice_i_plus_1 = strip.upper_slice_index
            
            vertex.tex_coord_0 = project_to_texture_coords(vertex, slice_i)
            vertex.tex_coord_1 = project_to_texture_coords(vertex, slice_i_plus_1)
            vertex.alpha = compute_alpha_factor(vertex, slice_i, slice_i_plus_1)
            
        # Step 4 & 5: Multi-texture blending
        # General Combiner 0: A=Texture0, B=PrimaryAlpha, C=Texture1, D=INVERT(B)
        # Output = A*B + C*D
        # Final Combiner: A=PrimaryColor(RGB), B=SecondaryColor, C=ZERO, D=ConstColor2
        # Output = A*B + C*D
        # (Hardware combiner setup is performed via standard OpenGL register combiner calls)
        draw_polygon_strip(strip)

def intersect_plane_bbox(plane, bbox):
    """Returns the convex polygon formed by intersecting a plane with an axis-aligned bounding box."""
    # Standard plane-box intersection algorithm (e.g., Sutherland-Hodgman or Liang-Barsky extension)
    pass

def find_intersecting_axial_slices(polygon, bbox):
    """Returns the indices of the axial texture slices that the polygon crosses."""
    # Determine min and max depth of polygon relative to volume axis
    pass

def clip_polygon_to_axial_strips(polygon, slice_indices):
    """Splits the polygon into strips bounded by adjacent axial slice planes."""
    # Clip polygon against each slice plane boundary
    pass

def project_to_texture_coords(vertex, slice_index):
    """Maps a 3D vertex position to 2D texture coordinates for a given axial slice."""
    pass

def compute_alpha_factor(vertex, slice_i, slice_i_plus_1):
    """Computes interpolation factor alpha: 0 at slice_i, 1 at slice_i_plus_1."""
    pass

def draw_polygon_strip(strip):
    """Renders the polygon strip with bound textures and configured combiners."""
    pass
```

**Parameters**
*   **Viewport Size:** Example implementation used $600 \times 600$ pixels [S1].

**Edge Cases and Pitfalls**
*   **Visual Artifacts:** Incorrect texture coordinate mapping or incorrect blending computation leads to visible artifacts, particularly when mapping texture slices at incorrect positions [S1].
*   **Correction Overhead:** Calculating correct texture coordinates is necessary to remove artifacts. Disabling this calculation increases performance by only $0.2$ frames per second for moderate datasets, indicating the overhead is negligible compared to the visual quality gain [S1].

## Operating characteristics
*   **Time Complexity:** Theoretical speedup of 2x compared to single-texture rendering due to halving the polygon count [S1].
*   **Memory Bandwidth:** The method requires accessing complete texture information for both slices in a single render pass, which can bottleneck performance [S1].
*   **Measured Performance:**
    *   Hardware: NVIDIA GeForce 256, 32 MB DDR, 500 MHz Pentium III [S1].
    *   The actual speedup observed was a factor of 1.8, not the theoretical 2.0, due to limited memory bandwidth [S1].
*   **Strengths:** Reduces polygon count by half; avoids 3D texture memory requirements; suitable for viewport-aligned slices [S1].
*   **Limitations:** Performance gains are capped by memory bandwidth; requires hardware support for register combiners (legacy OpenGL); sensitive to texture coordinate accuracy [S1].

## Background
This method was developed for interactive volume rendering on standard PC graphics hardware in the early 2000s [S1]. It was implemented on Windows NT with an NVIDIA GeForce 256 processor [S1]. The technique leverages the multi-texturing capabilities of early GPUs to perform linear interpolation between 2D texture slices, enabling efficient multi-planar reformatting (MPR) without the need for 3D texture units or complex ray-casting algorithms [S1].

## Sources
[S1] Interactive Volume Rendering on Standard PC Graphics Hardware, https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
