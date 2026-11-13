# Object Contour NPR

Also called: Csébfalvi's contour method, GPU-based contour tracing, Geometry shader contour rendering, Gradient-based contour visualization, View-Dependent Contour Generation

## Overview
Object Contour NPR generates view-dependent line primitives that demarcate areas where an object's surface turns away from the viewer, conveying shape through boundary emphasis [S1]. Unlike hatches, which are often pre-rendered to texture buffers, these contours are traced and rendered directly to the screen in real-time [S1]. It differs from direct volume rendering by relying solely on gradient magnitude and viewing angle rather than voxel intensity values, allowing for faster parameter tuning [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$, a viewing direction vector $\vec{v}$, and a user-defined distance threshold $d_{thresh}$ defining the neighborhood around the contour.
*   **Output:** A set of line strips rendered directly to the screen buffer.

**Data Structures:**
*   **Particle Set:** A collection of point primitives. Each particle $P_i$ stores its 3D position $(x, y, z)$ in the volume.
*   **Gradient Field:** A 3D vector field $\nabla V$ computed via 4D linear regression [S2, unofficial].

**Steps:**
1.  **Gradient Estimation:** Compute the gradient $\nabla V$ at every voxel. Conventional central difference is insufficient due to narrow neighborhood reliance; use 4D linear regression for accuracy [S2, unofficial].
2.  **Particle Selection:** Identify particles within the user-defined distance $d_{thresh}$ from the implicit contour surface. The contour is implicitly defined by the condition where the surface turns away from the viewer, characterized by locally high gradient values [S1][S2, unofficial].
3.  **Primitive Transformation:** Treat selected particles as point primitives. Pass them to a geometry shader [S1].
4.  **Contour Tracing:** The geometry shader traces along the contour direction, transforming point primitives into line strips that resemble contour segments [S1]. The trace direction and constant-width measure are adopted from the VolumeFlies framework [S1].
5.  **Rendering:** Render the generated line strips directly to the screen. Do not record them to a texture buffer [S1].
6.  **Color/Opacity Assignment:** For each voxel/pixel, determine color and opacity based only on the gradient magnitude $|\nabla V|$ and the angle $\theta$ between the viewing direction $\vec{v}$ and the gradient vector $\nabla V$ [S2, unofficial].

**Formulas:**
*   **Contour Condition:** The contour exists where the gradient magnitude is high and the surface normal turns away from the view.
*   **Viewing Angle:** $\cos \theta = \frac{\nabla V \cdot \vec{v}}{|\nabla V| |\vec{v}|}$ [S2, unofficial].
*   **VolumeFlies Measure:** The original framework uses a curvature-dependent measure $\tau$ with a threshold for constant-width contours [S1].

**Parameters:**
*   $d_{thresh}$: User-defined distance from the contour for selecting particles [S1].
*   Gradient estimation method: 4D linear regression [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Accuracy:** Using conventional central difference leads to inaccurate contours because it only considers a narrow voxel neighborhood [S2, unofficial].
*   **Performance:** Adding multiple particle sets for combined styles incurs a performance cost that may lead to loss of interactivity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_object_contours(volume, view_direction, distance_threshold):
    # Step 1: Compute gradients using 4D linear regression
    # Central difference is explicitly avoided [S2, unofficial]
    gradients = compute_gradient_4d_linear_regression(volume)
    
    # Step 2: Select particles near contours
    # Contours are areas where surface turns away from viewer [S1]
    # This implies high gradient magnitude and specific view-angle relationship
    particles = []
    for voxel in volume:
        grad = gradients[voxel]
        mag = magnitude(grad)
        # Implicit contour detection based on gradient and view angle [S2, unofficial]
        if is_near_contour(voxel, grad, view_direction, distance_threshold):
            particles.append(voxel.position)
            
    # Step 3 & 4: Geometry Shader Processing
    # Transform points to line strips [S1]
    line_strips = geometry_shader_trace(particles, gradients)
    
    # Step 5: Render directly to screen [S1]
    # No texture buffer recording
    render_to_screen(line_strips)
    
    # Step 6: Color/Opacity based on gradient and angle [S2, unofficial]
    for strip in line_strips:
        for point in strip:
            grad = gradients[point]
            angle_cos = dot(grad, view_direction) / (magnitude(grad) * magnitude(view_direction))
            color = map_gradient_and_angle(magnitude(grad), angle_cos)
            set_point_color(point, color)

def is_near_contour(position, gradient, view_dir, threshold):
    # Placeholder for specific contour proximity logic
    # Based on VolumeFlies curvature measure tau [S1]
    # and high gradient values [S2, unofficial]
    pass
```

## Subtopics

### Image Space Outline Generation
Generates outlines in image space rather than volume space. Choose when volume traversal is too expensive. [Image Space Outline Generation](788-image-space-outline-generation.md)

### Lasso Contour Rendering
Uses lasso-style strokes. Choose for artistic, hand-drawn aesthetics. [Lasso Contour Rendering](1855-lasso-contour-rendering.md)

### Curvature-Controlled Style Contours
Controls contour style via curvature. Choose when curvature detail is critical. [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md)

### Silhouette Rendering
Renders only the silhouette edge. Choose for simple boundary definition. [Silhouette Rendering](7959-silhouette-rendering.md)

### Topological Landscape
Maps topology to height. Choose for topological feature emphasis. [Topological Landscape](8481-topological-landscape.md)

### Boundary Enhancement Volume Style
Enhances boundaries with volume style. Choose for integrated volume-edge look. [Boundary Enhancement Volume Style](10863-boundary-enhancement-volume-style.md)

### Suggestive Contour Extraction
Extracts suggestive contours. Choose for hidden surface hints. [Suggestive Contour Extraction](13101-suggestive-contour-extraction.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of particles and the cost of 4D linear regression gradient estimation. Interactivity is maintained for single particle sets but degrades with multiple sets [S1].
*   **Memory Complexity:** Requires storage for the volume, gradient field, and particle set.
*   **Strengths:** Fast parameter tuning due to independence from voxel intensity values [S2, unofficial]. View-dependent accuracy [S1].
*   **Limitations:** High reliance on accurate gradient reconstruction [S2, unofficial]. Performance loss with multiple particle sets [S1].
*   **When to use:** When real-time, view-dependent boundary emphasis is needed and voxel intensity is not the primary data source.
*   **When not to use:** When voxel intensity-based classification is required or when pre-rendered hatches are sufficient.

## Background
Object Contour NPR was presented by Csébfalvi et al. in 2001 [S2, unofficial]. It was adopted from the VolumeFlies framework [BVvW07], which introduced constant-width contours using a threshold on a curvature-dependent measure $\tau$ [S1]. The method is part of the broader field of illustrative volume rendering, aiming to enhance perceptual clarity through non-photorealistic edge representation [G].

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
