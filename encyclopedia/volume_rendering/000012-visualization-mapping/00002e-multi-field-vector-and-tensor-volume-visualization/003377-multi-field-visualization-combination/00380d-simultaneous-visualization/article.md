# Simultaneous Visualization

Also called: Multi-property Visualization, Multivariate Visualization, Superimposed Volume and Stroke Rendering

## Overview
Simultaneous Visualization is a multidimensional visualization strategy that presents multiple data properties in a single image by combining distinct rendering techniques, such as superimposing volumetric rendering of a scalar field with stroke-based rendering of a vector field [S2]. It differs from [Multi-field Visualization Combination](7061-multi-field-visualization-combination.md) by explicitly leveraging non-photorealistic rendering (NPR) strokes to convey vector direction, whereas the parent category focuses more broadly on combining volumetric objects with geometric glyphs or streamlines [S1, unofficial][S2].

## How it works
**Inputs and Outputs**
- **Input:** Multivariate volume data containing at least two distinct properties (e.g., a scalar field like vorticity magnitude, temperature, or salinity, and a vector field like velocity) [S2].
- **Output:** A single composite image where volumetric structures and vector flow directions are perceptually integrated [S2].

**Data Structures**
- **Multivariate Data:** A 3D grid or unstructured mesh where each data point holds multiple scalar and vector properties [S2].
- **Stroke Representation:** A set of line segments defined by start points, end points, and attributes (color, opacity, width) [S2].

**Steps**
1. **Volumetric Rendering:** Render the primary scalar property (e.g., vorticity magnitude, temperature) using standard volume rendering techniques to establish structural context and object location [S2].
2. **Stroke Superposition:** Render the secondary vector property (e.g., velocity) as sparse strokes over the volumetric background to indicate local flow direction [S2].
3. **Boundary Highlighting:** Highlight specific boundaries (e.g., ocean surface) to aid in perceiving the volumetric depth and structure of the data [S2].
4. **Contextual Darkening:** Darken strokes in regions away from areas of interest (e.g., away from the sea surface) to enhance depth perception or focus attention [S2].
5. **Interactive Adjustment:** Allow the user to interactively control stroke density and size to balance detail and comprehension [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simultaneous_visualization(scalar_field, vector_field, boundary_surface):
    # Step 1: Render scalar field volumetrically [G]
    background = standard_volume_render(scalar_field) # Standard volume rendering algorithm [G]

    # Step 2: Generate strokes from vector field [G]
    strokes = []
    step = 5 # Default sampling step [S2]
    for x in range(0, scalar_field.width, step):
        for y in range(0, scalar_field.height, step):
            for z in range(0, scalar_field.depth, step):
                p = (x, y, z)
                v = vector_field[p]
                end = (p[0] + v[0], p[1] + v[1], p[2] + v[2])
                strokes.append({'start': p, 'end': end, 'opacity': 1.0, 'width': 2.0})

    # Step 3: Apply contextual darkening based on distance to boundary [S2]
    for s in strokes:
        mid = ((s['start'][0] + s['end'][0]) / 2, (s['start'][1] + s['end'][1]) / 2, (s['start'][2] + s['end'][2]) / 2)
        # Distance to boundary calculation not specified in sources [G]
        dist = compute_distance_to_boundary(mid, boundary_surface) # [G]
        s['opacity'] = max(0.0, 1.0 - dist * 0.1)

    # Step 4: Highlight boundaries to aid depth perception [S2]
    boundary_img = render_geometry(boundary_surface) # Standard geometry rendering [G]

    # Step 5: Composite strokes and boundaries over volumetric background [G]
    final = alpha_blend(background, boundary_img) # Standard compositing [G]
    for s in strokes:
        draw_line(final, s['start'], s['end'], color=(0, 0, 0), opacity=s['opacity'], width=s['width']) # Standard rasterization [G]
    return final
```

**Parameters**
- **Stroke Density:** Controls the sampling step size for stroke generation. Interactive control is recommended to avoid clutter [S2].
- **Darkening Factor:** Reduces stroke opacity for regions away from the area of interest (e.g., sea surface) to enhance depth perception [S2].
- **Boundary Highlight Threshold:** Not given in the sources [G].

**Edge Cases and Pitfalls**
- **Visual Clutter:** High stroke density can obscure the volumetric structure; interactive control is recommended to mitigate this [S2].
- **Depth Perception:** Without boundary highlighting or darkening, superimposing 2D strokes on 3D volume can flatten depth perception [S2].
- **Precision:** Not given in the sources [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources [G].
- **Memory Complexity:** Not given in the sources [G].
- **Strengths:** Enhances comprehension of vector fields (direction, motion) while maintaining structural context from scalar fields. Interactive control allows users to balance detail and clarity [S2].
- **Limitations:** Can be visually cluttered if not tuned. Relies on user interaction for optimal density settings [S2].
- **When to Use:** When both structural context (scalar) and flow/direction (vector) are critical, and traditional glyphs or streamlines alone are insufficient [S1, unofficial][S2].
- **When Not to Use:** Not given in the sources [G].

## Background
Simultaneous Visualization was investigated by Stompel et al. in the context of CFD and ocean simulation visualizations [S1, unofficial][S2]. It builds on earlier work by Joshi and Rheingans on time-varying data visualization, such as speedlines and opacity modulation [S1, unofficial]. The technique is particularly useful in disciplines like medical imaging (combining CT, MRI, PET) and CFD (combining streamlines and isosurfaces) where multiple modalities or properties need to be perceived simultaneously [S2].

## Sources
[S1] Generalized Framework and Algorithms for Illustrative Visualization of Time-Varying Data on Unstructured Meshes, https://digital.library.unt.edu/ark:/67531/metadc838861/?q=illustrative volume rendering, fetched 2026-10-28 (unofficial)
[S2] PDF Visualization of Multidimensional, Multivariate Volume Data Using ..., https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/pg-086_stompel_a_NEW.pdf, fetched 2026-10-28
