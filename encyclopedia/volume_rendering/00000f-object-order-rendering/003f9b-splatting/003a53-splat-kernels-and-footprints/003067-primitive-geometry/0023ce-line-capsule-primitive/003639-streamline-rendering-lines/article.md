# Streamline Rendering (Lines)

Also called: Stroke Based Rendering

## Overview
Streamline Rendering (Lines) visualizes vector fields by placing discrete geometric elements called strokes along streamlines, rather than drawing continuous lines, tubes, or ribbons [S2]. It is an automatic non-photorealistic rendering (NPR) approach designed to illustrate flow direction and field properties while avoiding the visual clutter and preprocessing costs associated with methods like Volume Line Integral Convolution (LIC) [S2]. It distinguishes itself from sibling splatting methods by using discrete stroke geometry to encode directionality and field magnitude, often rendered in a separate pass or as opaque geometry prior to volume rendering [S2].

## How it works
The method relies on generating streamlines from seed points and rendering them as discrete strokes. The implementation typically utilizes OpenGL Vertex Buffer Objects (VBOs) to store the geometry and attributes of these strokes.

### Inputs and Outputs
*   **Inputs**: Vector field data, seed point locations, user-defined parameters (stroke length, color mapping, density), and rendering context (z-buffer state).
*   **Outputs**: Rasterized image containing discrete strokes representing the flow field, composited with the underlying volume data.

### Data Structures
The geometry is stored in GPU memory using VBOs. The specific layout observed in CUDA-based implementations includes:
*   `m_verticesVbo`: An OpenGL `GL_ARRAY_BUFFER` storing line vertices.
    *   Type: `float3` (32-bit float per component).
    *   Size: `m_linesCount * m_maxGeometrySamples * sizeof(float3)` bytes.
*   `m_colorsVbo`: An OpenGL `GL_ARRAY_BUFFER` storing line colors.
    *   Type: `float3` (RGB).
    *   Size: `m_linesCount * m_maxGeometrySamples * sizeof(float3)` bytes.
*   [S1, unofficial]

### Steps
1.  **Seed Point Selection**: Select seed points based on an enhancement criterion specified by the user to avoid visual clutter and highlight regions of interest [S2].
2.  **Streamline Tracing**: Trace streamlines from the seed points through the vector field using traditional streamline tracing techniques [S2].
3.  **Stroke Generation**: Convert the traced streamlines into discrete strokes. Each stroke is a geometric primitive (e.g., a line segment or polygon) [S2].
4.  **Attribute Assignment**:
    *   **Direction**: Encode direction using a fading tail on the stroke [S2].
    *   **Color**: Map field properties (e.g., velocity magnitude) to the stroke color [S2].
5.  **Rendering**:
    *   Render the strokes as opaque, untextured geometry with the z-buffer enabled [S2].
    *   This can be done prior to rendering the volume as textured polygons, or in an additional third pass [S2].
    *   If rendered prior, the volume is rendered on top; if in a third pass, compositing rules apply [S2].

### Parameters
*   **Stroke Length**: Controls the trade-off between direction clarity and structural overview. Shorter strokes show direction information better; longer strokes reveal structural information better [S2].
*   **Saturation and Brightness**: Must be chosen carefully. High saturation/brightness in dense, intertwined color lines can obscure structural information [S2].
*   **Seed Density**: Interactive adjustment allows users to change the density of strokes to improve comprehension of the velocity field [S2].

### Pitfalls
*   **Visual Clutter**: Dense, intertwined color lines generally do not present structural information well [S2].
*   **Preprocessing Cost**: While traditional streamline tracing is chosen to avoid this, related methods like Volume LIC have preprocessing costs that are too costly for interactive exploration [S2].
*   **Color Choice**: Incorrect saturation and brightness levels can lead to poor visualization of the flow field structure [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_streamlines(vector_field, seed_points, params):
    """
    Renders streamlines as discrete strokes.
    params: dict containing 'stroke_length', 'color_map', 'render_order'
    """
    # 1. Trace streamlines
    streamlines = []
    for seed in seed_points:
        # Trace from seed through vector_field
        points = trace_streamline(seed, vector_field)
        streamlines.append(points)

    # 2. Generate stroke geometry
    vertices = []
    colors = []
    
    for line in streamlines:
        # Convert line points to discrete strokes
        # Apply fading tail for direction indication
        stroke_vertices = generate_stroke_geometry(line, params['stroke_length'])
        vertices.extend(stroke_vertices)
        
        # Color based on field property (e.g., velocity magnitude)
        # Carefully chosen saturation/brightness
        stroke_colors = map_color(line, vector_field, params['color_map'])
        colors.extend(stroke_colors)

    # 3. Upload to GPU VBOs
    # m_verticesVbo: GL_ARRAY_BUFFER, size = len(vertices) * sizeof(float3)
    # m_colorsVbo: GL_ARRAY_BUFFER, size = len(colors) * sizeof(float3)
    upload_vbo('m_verticesVbo', vertices)
    upload_vbo('m_colorsVbo', colors)

    # 4. Render
    if params['render_order'] == 'before_volume':
        # Render strokes as opaque untextured geometry with z-buffer on
        set_render_mode(opaque=True, textured=False, z_buffer=True)
        draw_vbo('m_verticesVbo', 'm_colorsVbo')
        # Then render volume as textured polygons
        render_volume()
    elif params['render_order'] == 'third_pass':
        # Render volume first
        render_volume()
        # Then render strokes in additional pass
        set_render_mode(opaque=True, textured=False, z_buffer=True)
        draw_vbo('m_verticesVbo', 'm_colorsVbo')
```

## Subtopics

### Stroke Based Rendering
An automatic non-photorealistic rendering (NPR) approach for illustrating vector fields by placing discrete elements called strokes, rather than drawing lines, tubes, or ribbons [S2]. It is chosen when traditional flow visualization methods (lines, tubes) are too cluttered or when artistic/NPR styles are desired [S2]. It sets itself apart by using discrete elements that can be styled (e.g., pencil drawing) and colored to indicate direction and magnitude [S2].

## Operating characteristics
*   **Time Complexity**: Dependent on the number of seed points and the length of streamlines traced. Interactive exploration is possible if seed selection and tracing are optimized, avoiding the high preprocessing cost of Volume LIC [S2].
*   **Memory Complexity**: Proportional to the number of strokes and their length. Stored in VBOs (`m_verticesVbo`, `m_colorsVbo`) [S1, unofficial].
*   **Strengths**: Avoids visual clutter of continuous lines; allows interactive adjustment of density and viewing angle; provides clear direction indication via fading tails; can be composited with volume data [S2].
*   **Limitations**: Dense color lines can obscure structure; requires careful tuning of stroke length and color saturation/brightness [S2].
*   **When to use**: When illustrating vector fields with NPR style, requiring clear direction and magnitude information without the clutter of tubes/ribbons, and when interactive exploration is needed [S2].
*   **When not to use**: When high preprocessing cost is acceptable and LIC is preferred for texture-like visualization, or when structural information is more critical than direction (longer strokes may help, but dense lines still pose a risk) [S2].

## Background
Streamline rendering using strokes was proposed as an alternative to conventional flow visualization methods (lines, tubes, ribbons) and Volume Line Integral Convolution (LIC) [S2]. Volume LIC was deemed too costly for interactive exploration due to preprocessing requirements [S2]. The method was developed to allow selective placement of seed points based on user-specified enhancement criteria, improving comprehension of velocity fields through interactive adjustment of stroke density and viewing angle [S2]. It has been used in simultaneous visualization of vorticity fields (via volume rendering) and velocity fields (via stroke-based rendering) [S2].

## Sources
[S1] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
[S2] PDF Visualization of Multidimensional, Multivariate Volume Data Using ..., https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/pg-086_stompel_a_NEW.pdf, fetched 2026-10-28
