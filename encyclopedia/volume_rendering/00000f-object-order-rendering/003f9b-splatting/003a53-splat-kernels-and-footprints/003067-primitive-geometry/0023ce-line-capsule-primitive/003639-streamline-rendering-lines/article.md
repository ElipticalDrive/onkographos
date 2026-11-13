# Streamline Rendering (Lines)


## Overview
Streamline Rendering (Lines) visualizes vector fields by placing discrete geometric elements called strokes along streamlines, rather than drawing continuous lines, tubes, or ribbons [S2]. It is an automatic non-photorealistic rendering (NPR) approach designed to illustrate flow direction and field properties [S2]. It distinguishes itself from sibling splatting methods by using discrete stroke geometry to encode directionality and field magnitude, often rendered in a separate pass or as opaque geometry prior to volume rendering [S2].

## How it works
The method generates streamlines from seed points and renders them as discrete strokes. The implementation typically utilizes OpenGL Vertex Buffer Objects (VBOs) to store the geometry and attributes of these strokes.

### Inputs and Outputs
*   **Inputs**: Vector field data, seed point locations, and user-defined parameters (stroke length, color mapping, density) [S2].
*   **Outputs**: Rasterized image containing discrete strokes representing the flow field, composited with the underlying volume data [S2].

### Data Structures
The geometry is stored in GPU memory using VBOs [S1, unofficial]. The specific layout includes:
*   `m_verticesVbo`: An OpenGL `GL_ARRAY_BUFFER` storing line vertices. Type: `float3`. Size: `m_linesCount * m_maxGeometrySamples * sizeof(float3)` bytes [S1, unofficial].
*   `m_colorsVbo`: An OpenGL `GL_ARRAY_BUFFER` storing line colors. Type: `float3`. Size: `m_linesCount * m_maxGeometrySamples * sizeof(float3)` bytes [S1, unofficial].

### Steps
1.  **Seed Point Selection**: Select seed points based on an enhancement criterion specified by the user to highlight regions of interest [S2].
2.  **Streamline Tracing**: Trace streamlines from the seed points through the vector field using traditional streamline tracing techniques [S2].
3.  **Stroke Generation**: Convert the traced streamlines into discrete strokes [S2].
4.  **Attribute Assignment**: Encode direction using a fading tail on the stroke [S2]. Map field properties (e.g., velocity magnitude) to the stroke color [S2].
5.  **Rendering**: Render the strokes as opaque, untextured geometry with the z-buffer enabled [S2]. This is done either prior to rendering the volume as textured polygons, or in an additional third pass [S2].

### Parameters
*   **Stroke Length**: Controls the trade-off between direction clarity and structural overview. Shorter strokes show direction information better; longer strokes reveal structural information better [S2].
*   **Saturation and Brightness**: Must be chosen carefully. High saturation/brightness in dense, intertwined color lines can obscure structural information [S2].
*   **Seed Density**: Interactive adjustment allows users to change the density of strokes to improve comprehension of the velocity field [S2].

### Pitfalls
*   **Visual Clutter**: Dense, intertwined color lines generally do not present structural information well [S2].
*   **Preprocessing Cost**: Related methods like Volume LIC have preprocessing costs that are too costly for interactive exploration, which is why traditional streamline tracing is chosen instead [S2].
*   **Color Choice**: Incorrect saturation and brightness levels can lead to poor visualization of the flow field structure [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_streamlines(vector_field, seed_points, params):
    # 1. Trace streamlines from seed points
    streamlines = []
    for seed in seed_points:
        points = [seed]
        current = seed
        for _ in range(params['max_steps']):
            direction = vector_field.sample(current)
            if direction.magnitude() == 0:
                break
            current = current + direction * params['step_size']
            points.append(current)
        streamlines.append(points)

    # 2. Generate stroke geometry and assign attributes
    vertices = []
    colors = []
    for line in streamlines:
        for i in range(len(line) - 1):
            p0 = line[i]
            p1 = line[i+1]
            vertices.append(p0)
            vertices.append(p1)
            # Encode direction using fading tail
            alpha = 1.0 - (i / len(line))
            # Map field property (e.g., velocity magnitude) to color
            mag = vector_field.magnitude(p0)
            r, g, b = params['color_map'](mag)
            colors.append((r, g, b, alpha))

    # 3. Store geometry in GPU memory using VBOs
    # m_verticesVbo: GL_ARRAY_BUFFER, size = len(vertices) * sizeof(float3)
    # m_colorsVbo: GL_ARRAY_BUFFER, size = len(colors) * sizeof(float4)
    gpu_vertices = [0.0] * (len(vertices) * 3)
    gpu_colors = [0.0] * (len(colors) * 4)
    for i, v in enumerate(vertices):
        gpu_vertices[i*3:i*3+3] = [v.x, v.y, v.z]
    for i, c in enumerate(colors):
        gpu_colors[i*4:i*4+4] = list(c)

    # 4. Render strokes as opaque untextured geometry with z-buffer on
    if params['render_order'] == 'before_volume':
        enable_depth_test()
        draw_line_segments(gpu_vertices, gpu_colors)
        render_volume_textured()
    elif params['render_order'] == 'third_pass':
        render_volume_textured()
        enable_depth_test()
        draw_line_segments(gpu_vertices, gpu_colors)

def enable_depth_test():
    # Standard graphics pipeline operation: glEnable(GL_DEPTH_TEST)
    pass

def draw_line_segments(vertices, colors):
    # Standard graphics pipeline operation: glDrawArrays(GL_LINES, ...)
    pass

def render_volume_textured():
    # Standard graphics pipeline operation: render volume as textured polygons
    pass
```

## Subtopics

### Stroke Based Rendering
An automatic non-photorealistic rendering (NPR) approach for illustrating vector fields by placing discrete elements called strokes, rather than drawing lines, tubes, or ribbons [S2]. It sets itself apart by using discrete elements that can be styled (e.g., pencil drawing) and colored to indicate direction and magnitude [S2]. It is chosen when traditional flow visualization methods (lines, tubes) are avoided in favor of NPR styles, and when interactive exploration is needed to avoid the high preprocessing cost of Volume LIC [S2].

## Operating characteristics
*   **Time Complexity**: Dependent on the number of seed points and the length of streamlines traced [G]. Interactive exploration is possible because traditional streamline tracing avoids the high preprocessing cost of Volume LIC [S2].
*   **Memory Complexity**: Proportional to the number of strokes and their length, stored in VBOs (`m_verticesVbo`, `m_colorsVbo`) [S1, unofficial].
*   **Strengths**: Allows interactive adjustment of density and viewing angle; provides clear direction indication via fading tails; can be composited with volume data [S2].
*   **Limitations**: Dense color lines can obscure structure; requires careful tuning of stroke length and color saturation/brightness [S2].
*   **When to use**: When illustrating vector fields with NPR style, requiring clear direction and magnitude information without the clutter of tubes/ribbons, and when interactive exploration is needed [S2].
*   **When not to use**: When high preprocessing cost is acceptable and LIC is preferred for texture-like visualization, or when structural information is more critical than direction (longer strokes may help, but dense lines still pose a risk) [S2].

## Background
Streamline rendering using strokes was proposed as an alternative to conventional flow visualization methods (lines, tubes, ribbons) and Volume Line Integral Convolution (LIC) [S2]. Volume LIC was deemed too costly for interactive exploration due to preprocessing requirements [S2]. The method was developed to allow selective placement of seed points based on user-specified enhancement criteria, improving comprehension of velocity fields through interactive adjustment of stroke density and viewing angle [S2]. It has been used in simultaneous visualization of vorticity fields (via volume rendering) and velocity fields (via stroke-based rendering) [S2].

## Sources
[S1] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
[S2] PDF Visualization of Multidimensional, Multivariate Volume Data Using ..., https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/pg-086_stompel_a_NEW.pdf, fetched 2026-10-28
