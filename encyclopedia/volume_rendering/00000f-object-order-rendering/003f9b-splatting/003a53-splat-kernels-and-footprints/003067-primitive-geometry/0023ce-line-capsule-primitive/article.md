# Line Capsule Primitive

Also called: Geometry Primitives and Mesh Shading, Mesh Lighting and Materials

## Overview
The Line Capsule Primitive is a geometric splatting method that renders line-based volumetric data as rotationally symmetric capsules in screen space [S1, unofficial]. It distinguishes itself from sibling methods like Circle Splat Rendering and Tetrahedral Splatting by explicitly modeling line segments as capsules with deficit-rule joints to maintain visual continuity without gaps or double brightness [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of line segments defined by endpoints in volume space, along with associated attributes (e.g., width, color, opacity) [S1, unofficial].
*   **Output:** A 2D image where lines are rendered as opaque or semi-transparent capsules, composited with other splats or background geometry [S1, unofficial].

**Data Structures:**
The primitive is defined by a line segment in 3D space [S1, unofficial]. In screen space, it is represented as a capsule, which is the Minkowski sum of a line segment and a disk [S1, unofficial].
*   **Segment:** Defined by two 3D vertices $\mathbf{v}_1, \mathbf{v}_2$ [S1, unofficial].
*   **Capsule Profile:** A rotationally symmetric function of the 2D point-to-segment distance in screen space [S1, unofficial].
*   **Joint Handling:** Corners between segments are composed using a deficit rule to prevent double brightness or gaps [S1, unofficial].

**Steps:**
1.  **Transformation:** Transform line segment endpoints from volume space to screen space using the Model-View-Projection (MVP) matrix [S1, unofficial].
2.  **Primitive Selection:** Determine whether to render the line as a **Capsule** or a **Screen-Space Quad** based on the effective segment load [S1, unofficial].
    *   Calculate effective segment load: $L = N_{seg} \times W_{max}$, where $N_{seg}$ is the authored segment count and $W_{max}$ is the on-screen width of the widest segment at the opening framing [S1, unofficial].
    *   If $L < 2,000,000$, use the Capsule primitive [S1, unofficial].
    *   If $L \ge 2,000,000$, use the Screen-Space Quad primitive [S1, unofficial].
    *   This policy can be pinned to either primitive type if desired [S1, unofficial].
3.  **Rasterization (Capsule):** Project the line segment into screen space, generate a rotationally symmetric footprint based on the 2D point-to-segment distance, and apply the deficit rule at joints to ensure correct brightness and no gaps [S1, unofficial].
4.  **Rasterization (Quad):** Generate a screen-space quad covering the bounding box of the projected segment [S1, unofficial].
5.  **Compositing:** Blend the capsule or quad contributions into the frame buffer [S1, unofficial].

**Formulas:**
*   **Effective Segment Load:**
    $$ L = N_{seg} \times W_{max} $$
    Where:
    *   $N_{seg}$: Authored segment count.
    *   $W_{max}$: On-screen width of the widest segment at the opening framing [S1, unofficial].
*   **Capsule Footprint:**
    The intensity profile $I(x,y)$ at screen coordinates $(x,y)$ is determined by the distance $d$ from $(x,y)$ to the projected line segment:
    $$ I(x,y) = f(d(x,y)) $$
    Where $d(x,y)$ is the 2D point-to-segment distance in screen space, and $f$ is the rotationally symmetric profile function [S1, unofficial].

**Parameters:**
*   **Effective Segment Load Threshold:** 2,000,000 [S1, unofficial].
*   **Joint Composition Rule:** Deficit rule [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Near-Axial Cases:** The capsule primitive matches or beats the exact volumetric primitive (closed-form convolution of a segment with a 3D Gaussian, integrated along the view ray) in visual comparisons, including near-axial cases [S1, unofficial].
*   **Volumetric Blending:** The primitive refuses volumetric blending because a triangle or line has zero thickness, providing no path length for absorption [S1, unofficial].
*   **Additive Ladder:** An additive ladder over arbitrary face order is refused because a prefix of such an order is a surface with holes, not a coarser surface [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_line_capsule(segments, camera_matrix, threshold=2000000):
    # 1. Transform segments to screen space
    screen_segments = []
    for seg in segments:
        v1_screen = project_to_screen(seg.v1, camera_matrix)
        v2_screen = project_to_screen(seg.v2, camera_matrix)
        screen_segments.append((v1_screen, v2_screen))

    # 2. Calculate effective segment load
    n_seg = len(screen_segments)
    w_max = 0.0
    for v1, v2 in screen_segments:
        width = distance(v1, v2)
        if width > w_max:
            w_max = width

    effective_load = n_seg * w_max

    # 3. Select primitive
    if effective_load < threshold:
        primitive_type = "CAPSULE"
    else:
        primitive_type = "QUAD"

    # 4. Rasterize
    for v1, v2 in screen_segments:
        if primitive_type == "CAPSULE":
            # Rasterize rotationally symmetric footprint based on 2D point-to-segment distance
            # Apply deficit rule at joints to prevent double brightness or gaps
            rasterize_capsule_footprint(v1, v2)
        else:
            # Rasterize screen-space quad covering bounding box of projected segment
            rasterize_screen_quad(v1, v2)

    return frame_buffer

def project_to_screen(vertex, camera_matrix):
    # Standard MVP projection: multiply vertex by camera matrix, divide by w
    projected = [sum(camera_matrix[i][j] * vertex[j] for j in range(4)) for i in range(4)]
    return (projected[0] / projected[3], projected[1] / projected[3])

def distance(p1, p2):
    return ((p1[0] - p2[0])**2 + (p1[1] - p2[1])**2)**0.5

def rasterize_capsule_footprint(v1, v2):
    # Compute 2D point-to-segment distance for each pixel in bounding box
    # Write intensity I(x,y) = f(d(x,y)) to frame_buffer
    # Apply deficit rule at segment joints
    pass # Source does not provide exact rasterization implementation

def rasterize_screen_quad(v1, v2):
    # Compute bounding box of v1 and v2
    # Fill quad with primitive color/opacity to frame_buffer
    pass # Source does not provide exact rasterization implementation

frame_buffer = [] # Accumulated pixel values
```

## Subtopics

### [Streamline Rendering (Lines)](7718-streamline-rendering-lines.md)
Streamline Rendering (Lines) visualizes vector fields by placing discrete geometric elements called strokes along streamlines, rather than drawing continuous lines, tubes, or ribbons. It is an automatic non-photorealistic rendering (NPR) approach designed to illustrate flow direction and field properties while avoiding the visual clutter and preprocessing costs associated with methods like Volume Line Integral Convolution (LIC). It distinguishes itself from sibling splatting methods by using discrete stroke geometry to encode directionality and field magnitude, often rendered in a separate pass or as opaque geometry prior to volume rendering.

### Vector Text Rendering via Strokes
Renders text by mapping each character to a set of 2D line segments (strokes) in a 1x1 unit square, then transforming them to 3D space and drawing as capsules [S2, unofficial]. Supports 6 orientations and cursor offset for spacing [S2, unofficial]. Character stroke definitions are hardcoded as line segments in a 1x1 grid [S2, unofficial]. The `TextGeometry` data structure contains `size` (float), `thickness` (float), `pos` (Vec3), `orientation` (enum), and `text` (string) [S2, unofficial]. `TextOrientation` is an enum with values `POS_Z`, `NEG_Z`, `POS_X`, `NEG_X`, `POS_Y`, `NEG_Y`, and `default` [S2, unofficial]. The rendering loop initializes `cursorOffset = 0.0f`, iterates over each character `c`, converts to uppercase, maps 2D stroke coordinates `(x1,y1)` to `(x2,y2)` using an `addLine` lambda, applies scale `s`, offset `cursorOffset`, and orientation transform via `mapCoords`, creates `CapsuleGeometry`, calls `drawCapsule`, and increments `cursorOffset += 1.5f` [S2, unofficial].

## Operating characteristics
*   **Performance:** The capsule primitive is preferred over the exact volumetric primitive (closed-form convolution of a segment with a 3D Gaussian, integrated along the view ray) because it matched or beat it in blinded visual comparisons while costing 2.7–5x less than the screen-space quad per frame [S1, unofficial].
*   **Hardware Differences:** The performance difference between capsule and quad primitives is 1.04–1.11x on Apple GPUs and ~1.5x on thin lines on discrete NVIDIA GPUs [S1, unofficial].
*   **Memory:** Meshes (the only shaded geometry type in the same system) contribute no size padding to scene bounds and are loaded whole, not through a spatial index [S1, unofficial].
*   **Strengths:** Efficient rasterization of line data; handles joints correctly without gaps or double brightness; supports physically based materials and transmission for meshes [S1, unofficial].
*   **Limitations:** Cannot perform volumetric blending (absorption) due to zero thickness; additive ladder over arbitrary face order is not supported [S1, unofficial].

## Background
The Line Capsule Primitive is part of the "Geometry primitives and mesh shading" system described in the Luxar viewer [S1, unofficial]. It was developed to provide scalable interactive web visualization of multidimensional scientific data, specifically for microscopy and other fields requiring efficient rendering of line-based structures [S1, unofficial]. The method was evaluated against exact volumetric primitives and screen-space quads, with the capsule primitive emerging as the preferred choice for most cases due to its balance of visual quality and performance [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
