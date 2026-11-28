# Hybrid Scanline A-Buffer Volume Rendering
Also called: Scanline A-Buffer with Volume Fragments

## Overview
Hybrid Scanline A-Buffer Volume Rendering integrates surface-defined geometry and procedural volume data (such as gases) into a single image by combining a fast scanline a-buffer algorithm for solids with volume rendering for translucent media [S1, unofficial]. It distinguishes itself from pure ray casting by using an a-buffer to manage fragment overlap and from pure polygon rendering by explicitly handling volumetric opacity and attenuation through sorted volume fragments [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A scene containing surface-defined objects (polygons) and volume-defined objects (procedural gases or fluids), a camera view, and lighting parameters [S1, unofficial].
*   **Outputs:** A final 2D image with correct blending of surfaces and volumes [S1, unofficial].

**Data Structures:**
*   **A-Buffer:** A data structure associated with a scanline that maintains a list for each pixel containing all fragments that partially or fully cover that pixel [S1, unofficial].
*   **Volume Fragments:** Sub-sections of the volume data that cover the current scanline, treated as fragments with average Z-depth values [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize A-Buffer:** Create an a-buffer for the current scanline containing a list for each pixel to hold fragments that partially or fully cover the pixel [S1, unofficial].
2.  **Process Surface Geometry:** Render surface-defined objects into the a-buffer by storing their fragments in the pixel lists based on coverage and depth [S1, unofficial].
3.  **Process Volumes:** For each volume object that intersects the current scanline, perform volume rendering calculations for the sections covering the scanline and generate a-buffer fragments for those sections [S1, unofficial]. Cease volume rendering for a specific pixel once full coverage of that pixel is achieved by either volume or surface-defined elements [S1, unofficial].
4.  **Sort Fragments:** Sort the volume a-buffer fragments into the a-buffer fragment list for each pixel based on their average Z-depth values [S1, unofficial].
5.  **Composite:** Render the final a-buffer fragment list for each pixel to produce the final color by combining the contributions of all sorted fragments [S1, unofficial].

**Illumination and Shadows:**
*   The system uses a physically-based low-albedo illumination and atmospheric attenuation model for gases [S1, unofficial].
*   Volumetric shadows are combined efficiently using three-dimensional shadow tables [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_scanline_a_buffer_rendering(scene, scanline_y):
    # 1. Initialize A-Buffer for the scanline
    a_buffer = {pixel_id: [] for pixel_id in range(scene.width)}
    
    # 2. Render surface-defined objects
    for surface in scene.surfaces:
        if surface.intersects_scanline(scanline_y):
            for frag in surface.get_fragments(scanline_y):
                a_buffer[frag.pixel_id].append(frag)
                
    # 3. Render volume-defined objects
    for volume in scene.volumes:
        if volume.intersects_scanline(scanline_y):
            for section in volume.get_sections(scanline_y):
                for v_frag in volume.render_section(section, scanline_y):
                    if not is_pixel_fully_covered(a_buffer[v_frag.pixel_id]):
                        a_buffer[v_frag.pixel_id].append(v_frag)
                        
    # 4. Sort fragments by average Z-depth for each pixel
    for pixel_id, fragments in a_buffer.items():
        fragments.sort(key=lambda f: f.average_z_depth)
        
    # 5. Composite final image
    image = {}
    for pixel_id, fragments in a_buffer.items():
        image[pixel_id] = composite_fragments(fragments)
    return image

def is_pixel_fully_covered(fragments):
    # [G] Check if accumulated coverage sum >= 1.0 or 
    # if opaque surface fragments exist at front-most depth.
    coverage_sum = sum(f.coverage for f in fragments)
    if coverage_sum >= 1.0:
        return True
    for f in fragments:
        if f.is_opaque and f.is_surface:
            return True
    return False

def composite_fragments(fragments):
    # [G] Standard front-to-back volume rendering composition
    # using sorted fragments by average Z-depth.
    # Illumination and shadow application details are not given in the sources [S1, unofficial].
    color = (0.0, 0.0, 0.0)
    alpha = 0.0
    for frag in fragments:
        c = frag.color
        a = frag.opacity
        color = color + c * a * (1.0 - alpha)
        alpha = alpha + a * (1.0 - alpha)
        if alpha >= 1.0:
            break
    return color
```

## Subtopics

### [Scanline A-buffer rendering](12488-scanline-a-buffer-rendering.md)
A rendering algorithm for surface-defined objects in hybrid scenes that creates an a-buffer for a scanline containing a list for each pixel of all fragments that partially or fully cover the pixel. It sets itself apart by explicitly managing fragment lists per pixel to support hybrid volume rendering, unlike standard scanline renderers that discard depth information early.

### Giertsen's scan-plane buffer algorithm
A hybrid volume rendering method that traverses the volume in object order and accumulates contributions in image order (object-then-image traversal) using a 2D scan-plane buffer to store information perpendicular to a scanline [S2]. It discretizes the z-dimension due to the scan-plane buffer [S2]. The algorithm processes scanlines from top to bottom and determines the intersections of the cells with the respective scan-plane in an incremental manner using a list of active cells whose y-extents cover the current scanline [S2]. The volume elements intersecting the current scan-plane are sliced by finding the edge intersections of faces of volume elements with the current scan-plane [S2]. Then, each slice is divided into triangles and each triangle is further decomposed into line segments in the z direction [S2]. The composition is carried out by processing the line segments in front-to-back order and linearly interpolating them along the ray [S2]. The quality of the images and the performance of the algorithm are heavily dependent on the discretization level of the scan-plane buffer [S2].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Described as a "fast" algorithm for surface-defined objects [S1, unofficial].
*   **Strengths:** Efficiently combines surface and volume rendering and uses 3D shadow tables for efficient volumetric shadowing [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When visualizing procedural gases or fluids alongside surface-defined objects in a hybrid scene [S1, unofficial].

## Background
The method was developed by David Ebert as part of a system for procedural modeling, animation, and rendering of gases, fluids, and textures [S1, unofficial]. It builds upon work in solid texturing and procedural noise functions (such as Perlin’s turbulence) [S1, unofficial]. The approach was presented in the context of visualizing computational fluid dynamics and hybrid scenes [S1, unofficial].

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
[S2] Direct volume rendering of unstructured grids, https://doi.org/10.1016/s0097-8493(03)00034-7, fetched 2026-10-24
