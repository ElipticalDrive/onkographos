# Bounding Wall Contour Overlay
Also called: Overlay Reference Field as Contours on Bounding Walls, Reference Field Contours on Bounding Walls, X13 --overlay

## Overview
Bounding Wall Contour Overlay is a non-photorealistic visualization technique that displays line contours of a reference scalar field on the vertical bounding walls of a 3D volume [S1, unofficial]. It solves the problem of providing spatial context for deformed interfaces without occluding the primary data or introducing visual clutter [S1, unofficial]. Unlike sibling methods that render edges within the volume or in image space, this method projects 2D contour lines onto the 3D scene's boundary planes, ensuring the annotations are read as context rather than competing data surfaces [S1, unofficial]. It builds on [Outline and Contour Generation](4137-outline-and-contour-generation.md).

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar field (the reference field, e.g., $b$) and a 3D rendering context [S1, unofficial].
- Output: A rendered image where line contours of the reference field are drawn on the two vertical walls of the volume's bounding box [S1, unofficial].

**Data Structures:**
- The reference field is a 3D grid of scalar values [S1, unofficial].
- The bounding box provides two vertical planes aligned with the primary axes [S1, unofficial].

**Steps:**
1. Identify the two vertical walls of the 3D volume's bounding box [S1, unofficial].
2. Extract the 2D slices of the reference field corresponding to these vertical planes [S1, unofficial].
3. Compute the 2D contour lines for each slice [G].
4. Render the contour lines as strokes on the identified walls using a transformation mechanism that maps 2D coordinates to the 3D plane [S1, unofficial].

**Parameters:**
- `--overlay-style`: Controls the rendering mode. The default is the wall contour overlay. A value of `iso` retains the isosurface version for comparison [S1, unofficial].
- `transformation`: The mechanism used to apply the 2D contour recipe onto the 3D plane [S1, unofficial].

**Pitfalls and Edge Cases:**
- **Isosurfaces inside the box:** Rendering isosurfaces of the reference field within the volume results in muddy visuals if the panel's midpoint color (e.g., grey for `:balance`) matches the isosurface color, causing it to be misinterpreted as washed-out data rather than context [S1, unofficial].
- **Occlusion by saturated hues:** Rendering isosurfaces with a saturated hue off the main axis makes them legible as context but occludes the primary data [S1, unofficial]. Transparency cannot fix this because some rendering engines (e.g., GLMakie) ignore alpha values on volume contours [S1, unofficial].
- **Alpha Ignorance:** In GLMakie, setting `--overlay-alpha` to 0.05 and 1.0 produces byte-identical renders, confirming that alpha transparency is ignored for volume contours [S1, unofficial].
- **Filled surfaces on walls:** Rendering the reference field as a filled surface on bounding walls can cause the floor to appear solid black if the colormap bottoms out at black, which is misinterpreted as a rendering fault [S1, unofficial]. Additionally, if the field carries no information (e.g., $b$ at fixed $z$ is nearly uniform), the filled surface is misleading [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_bounding_wall_contours(reference_field, bounding_box, scene):
    """
    Renders line contours of the reference field on the vertical walls of the bounding box.
    """
    # 1. Identify the two vertical walls of the bounding box
    # Assuming bounding_box defines extents along x, y, z
    # Vertical walls are at the minimum and maximum extents of the horizontal axis
    wall_indices = [0, reference_field.shape[2] - 1]
    
    for idx in wall_indices:
        # 2. Extract the 2D slice of the reference field corresponding to the wall
        field_slice = reference_field[:, :, idx]
        
        # 3. Compute contour lines for the 2D slice
        # Standard marching squares algorithm to extract iso-lines from a 2D grid
        # (Not given in the sources; implemented as a standard 2D contour extraction)
        contours = []
        for level in reference_field.levels:
            contours.extend(marching_squares(field_slice, level))
        
        # 4. Render the contours on the 3D wall using a transformation
        # 'transformation' maps the 2D contour data to the 3D plane
        # This applies a 2D rendering recipe onto a plane within the 3D scene
        for line in contours:
            # Map 2D (y, z) coordinates to 3D space at the wall's x-index
            x_coords = [idx] * len(line.y)
            draw_line_segments(scene, x_coords, line.y, line.z, style='line')
```

## Operating characteristics
- **Strengths:** Lines read as annotation rather than as a competing surface, keeping the deformed interface legible behind every panel [S1, unofficial]. It provides context without occluding the primary data [S1, unofficial].
- **Limitations:** Requires a rendering engine that supports projecting 2D recipes onto 3D planes [S1, unofficial]. Not suitable if the reference field carries no information on the walls (e.g., uniform at fixed $z$) [S1, unofficial].
- **When to use:** When providing spatial context for deformed interfaces without occluding the primary data is critical [S1, unofficial].
- **When not to use:** When the rendering engine does not support the required transformation mechanism or when the reference field is uniform on the bounding walls [S1, unofficial].

## Background
This technique was developed as part of the X13 batch render job for 3D isosurface figures, specifically to address visual clutter and occlusion issues in scientific visualization [S1, unofficial]. It was implemented using the Makie visualization library, specifically GLMakie [S1, unofficial]. The method emerged from iterative testing of isosurfaces and filled surfaces, which proved problematic due to color matching, occlusion, and alpha handling limitations in GLMakie [S1, unofficial].

## Sources
[S1] 3D isosurface figures: X13, a batch render job, and a subsetter, https://github.com/tomchor/CoarseGrainedKHAPE/pull/78, fetched 2026-10-28 (unofficial)
