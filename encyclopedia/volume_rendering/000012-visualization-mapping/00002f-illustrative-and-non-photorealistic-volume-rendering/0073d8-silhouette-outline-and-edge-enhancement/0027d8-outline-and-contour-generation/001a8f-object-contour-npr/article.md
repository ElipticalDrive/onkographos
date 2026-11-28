# Object Contour NPR

Also called: Csébfalvi's contour method, GPU-based contour tracing, Geometry shader contour rendering, Gradient-based contour visualization, View-Dependent Contour Generation

## Overview
Object Contour NPR generates view-dependent line primitives that demarcate areas where an object's surface turns away from the viewer, conveying shape through boundary emphasis [S1]. Unlike hatches, which are recorded to a texture buffer, these contours are traced and rendered directly to the screen [S1]. It differs from direct volume rendering by relying solely on gradient magnitude and viewing angle rather than voxel intensity values, allowing for faster parameter tuning [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$, a viewing direction vector $\vec{v}$, and a user-defined distance threshold $d_{thresh}$ defining the neighborhood around the contour [S1][S2, unofficial].
*   **Output:** A set of line strips rendered directly to the screen buffer [S1].

**Data Structures:**
*   **Particle Set:** A collection of point primitives [S1]. Each particle $P_i$ stores its 3D position $(x, y, z)$ in the volume [G].
*   **Gradient Field:** A 3D vector field $\nabla V$ computed via 4D linear regression [S2, unofficial].

**Steps:**
1.  **Gradient Estimation:** Compute the gradient $\nabla V$ at every voxel using 4D linear regression, as conventional central difference is insufficient due to its narrow voxel neighborhood reliance [S2, unofficial].
2.  **Particle Selection:** Identify particles within the user-defined distance $d_{thresh}$ from the implicit contour surface [S1].
3.  **Primitive Transformation:** Treat selected particles as point primitives [S1].
4.  **Contour Tracing:** Pass points to a geometry shader to trace along the contour direction, transforming them into line-strips that resemble contour segments [S1]. The trace direction and constant-width measure are adopted from the VolumeFlies framework [S1].
5.  **Rendering:** Render the generated line-strips directly to the screen, not to a texture buffer [S1].
6.  **Color/Opacity Assignment:** Determine color and opacity for each voxel/pixel based only on the gradient magnitude $|\nabla V|$ and the angle $\theta$ between the viewing direction $\vec{v}$ and the gradient vector $\nabla V$ [S2, unofficial].

**Formulas:**
*   **Contour Condition:** The contour exists where the gradient magnitude is high and the surface normal turns away from the view [S1][S2, unofficial].
*   **Viewing Angle:** $\cos \theta = \frac{\nabla V \cdot \vec{v}}{|\nabla V| |\vec{v}|}$ [G].
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
import math

def render_object_contours(volume, view_direction, distance_threshold):
    # Step 1: Gradient Estimation using 4D linear regression [S2, unofficial]
    # Central difference is explicitly avoided due to narrow neighborhood inaccuracy [S2, unofficial]
    gradients = compute_gradients_4d_linear_regression(volume)

    # Step 2: Particle Selection within user-defined distance [S1]
    particles = []
    for voxel in volume:
        grad = gradients[voxel]
        mag = math.sqrt(sum(c**2 for c in grad))
        # Contour condition: high gradient magnitude and surface turns away from view [S1][S2, unofficial]
        if is_near_contour(voxel, grad, view_direction, mag, distance_threshold):
            particles.append(voxel.position)

    # Step 3 & 4: Primitive Transformation and Contour Tracing [S1]
    # Geometry shader transforms point primitives into line-strips resembling contour parts
    # Trace direction and constant-width measure adopted from VolumeFlies [S1]
    line_strips = geometry_shader_trace(particles, gradients)

    # Step 5: Rendering directly to screen, not to a texture buffer [S1]
    render_to_screen(line_strips)

    # Step 6: Color/Opacity Assignment based on gradient magnitude and viewing angle [S2, unofficial]
    for strip in line_strips:
        for point in strip:
            grad = gradients[point]
            angle_cos = dot(grad, view_direction) / (math.sqrt(sum(c**2 for c in grad)) * math.sqrt(sum(c**2 for c in view_direction))) # [G]
            color = map_gradient_and_angle(math.sqrt(sum(c**2 for c in grad)), angle_cos)
            set_point_color(point, color)

# Helper functions (standard operations or marked if missing)
def compute_gradients_4d_linear_regression(volume):
    # 4D linear regression over voxel neighborhood [S2, unofficial]
    # Not given in the sources.
    pass

def is_near_contour(voxel, gradient, view_dir, mag, threshold):
    # Proximity check to implicit contour surface [S1]
    # Not given in the sources.
    pass

def geometry_shader_trace(particles, gradients):
    # Traces along contour direction, transforms points to line-strips [S1]
    # Not given in the sources.
    pass

def render_to_screen(line_strips):
    # Direct screen rendering [S1]
    # Not given in the sources.
    pass

def map_gradient_and_angle(mag, angle_cos):
    # Maps gradient magnitude and viewing angle to color/opacity [S2, unofficial]
    # Not given in the sources.
    pass

def set_point_color(point, color):
    # Assigns color to point primitive [S2, unofficial]
    # Not given in the sources.
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
*   **Memory Complexity:** Requires storage for the volume, gradient field, and particle set [S2, unofficial].
*   **Strengths:** Fast parameter tuning due to independence from voxel intensity values [S2, unofficial]. View-dependent accuracy [S1].
*   **Limitations:** High reliance on accurate gradient reconstruction [S2, unofficial]. Performance loss with multiple particle sets [S1].
*   **When to use:** When real-time, view-dependent boundary emphasis is needed and voxel intensity is not the primary data source [S1][S2, unofficial].
*   **When not to use:** When voxel intensity-based classification is required or when pre-rendered hatches are sufficient [S1][S2, unofficial].

## Background
Object Contour NPR was presented by Csébfalvi et al. in 2001 [S2, unofficial]. It was adopted from the VolumeFlies framework [BVvW07], which introduced constant-width contours using a threshold on a curvature-dependent measure $\tau$ [S1]. The method is part of the broader field of illustrative volume rendering, aiming to enhance perceptual clarity through non-photorealistic edge representation [G].

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
