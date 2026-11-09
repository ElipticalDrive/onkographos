# Object Contour NPR

Also called: Csébfalvi's contour method, GPU-based contour tracing, Geometry shader contour rendering, Gradient-based contour visualization, View-Dependent Contour Generation

## Overview
Object Contour NPR generates view-dependent lines that demarcate areas where an object's surface turns away from the viewer, conveying shape by emphasizing boundaries [S1]. Unlike hatches, which are often pre-rendered to texture buffers, these contours are traced directly to the screen using geometry shaders [S1]. It differs from standard Direct Volume Rendering (DVR) by relying exclusively on gradient magnitude and viewing angle rather than voxel intensity values, allowing for faster parameter tuning [S2, unofficial].

## How it works
**Inputs:**
*   Volume dataset $V$.
*   Viewing direction vector $\mathbf{v}$.
*   User-defined distance threshold $d_{thresh}$ for particle selection.
*   Gradient estimation method (recommended: 4D linear regression [S2, unofficial]).

**Outputs:**
*   Line strips rendered directly to the screen buffer.

**Data Structures:**
*   **Particle Set:** A collection of points sampled within the volume. Each particle $p_i$ has position $\mathbf{p}_i$ and associated gradient $\nabla V(\mathbf{p}_i)$.
*   **Gradient Field:** Estimated per particle. Conventional central difference is insufficient due to narrow neighborhood reliance [S2, unofficial]. 4D linear regression is required for accuracy [S2, unofficial].

**Steps:**
1.  **Gradient Estimation:** Compute the gradient $\nabla V$ at each particle position using 4D linear regression [S2, unofficial]. Let $\mathbf{g} = \nabla V(\mathbf{p})$.
2.  **Contour Detection:** Identify particles near contours. A particle is considered near a contour if the angle $\theta$ between the viewing direction $\mathbf{v}$ and the gradient $\mathbf{g}$ satisfies specific criteria, or if the gradient magnitude $|\mathbf{g}|$ is locally high [S1][S2, unofficial]. The original VolumeFlies framework uses a curvature-dependent measure $\tau$ with a threshold [S1].
3.  **Particle Selection:** Select all particles within a user-defined distance $d_{thresh}$ from the detected contour lines [S1].
4.  **Geometry Shader Tracing:** Pass selected particles as point primitives to a geometry shader. The shader traces along the contour direction, transforming points into line strips [S1].
5.  **Rendering:** Render the generated line strips directly to the screen. Do not record to a texture buffer [S1].
6.  **Color/Opacity Assignment:** For any voxel or particle, determine color and opacity based solely on $|\mathbf{g}|$ and the angle $\alpha$ between $\mathbf{v}$ and $\mathbf{g}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def object_contour_npr(volume, view_direction, distance_threshold):
    # 1. Estimate gradients using 4D linear regression [S2, unofficial]
    # Central difference is inaccurate [S2, unofficial]
    gradients = estimate_gradients_4d_linear_regression(volume)
    
    # 2. Identify contour particles
    # Criteria: High gradient magnitude and specific angle to view direction [S2, unofficial]
    # Or curvature-dependent measure tau threshold [S1]
    contour_particles = []
    for p in volume.particles:
        g = gradients[p]
        angle = acos(dot(normalize(g), normalize(view_direction)))
        # Simplified logic based on notes: select if near contour
        if is_near_contour(p, g, view_direction, distance_threshold):
            contour_particles.append(p)
            
    # 3. Transform to line strips via Geometry Shader [S1]
    # The shader handles the tracing logic
    line_strips = geometry_shader_trace(contour_particles)
    
    # 4. Render directly to screen [S1]
    render_to_screen(line_strips)
    
    return line_strips

def is_near_contour(p, g, view_direction, distance_threshold):
    # Note: Exact formula for 'distance from contour' is not explicitly defined 
    # in the provided text, but implies spatial proximity to the boundary defined by gradient/view angle [S1].
    # [G] Assuming a check on gradient magnitude and view angle alignment.
    return abs(dot(normalize(g), normalize(view_direction))) < 0.1 # Placeholder for specific threshold logic
```

**Parameters:**
*   $d_{thresh}$: User-defined distance from the contour for selecting particles [S1].
*   Gradient Method: 4D linear regression [S2, unofficial].

**Edge Cases:**
*   **Gradient Accuracy:** Using conventional central difference leads to inaccurate results because it only considers a narrow voxel neighborhood [S2, unofficial].
*   **Performance:** Adding multiple particle sets for combined styles increases performance cost, potentially losing interactivity [S1].

## Operating characteristics
*   **Time Complexity:** Interactive framerates are maintained for single styles, but performance degrades with multiple particle sets [S1].
*   **Memory:** Requires storage for particles and gradients. No texture buffer storage for contours [S1].
*   **Strengths:** Fast parameter tuning due to independence from voxel intensity values [S2, unofficial]. View-dependent accuracy [S1].
*   **Limitations:** Heavy reliance on accurate gradient reconstruction [S2, unofficial]. Not suitable for pre-rendering [S1].

## Background
Originated from Csébfalvi et al. (2001) [S2, unofficial]. Adopted from the VolumeFlies framework [BVvW07], which introduced constant-width contours using a threshold on a curvature-dependent measure $\tau$ [S1]. It is used in illustrative volume visualization to provide rough estimations of physical boundaries [S2, unofficial].

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
