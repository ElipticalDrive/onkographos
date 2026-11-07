# Object Contour NPR

Also called: Csébfalvi's contour method, GPU-based contour tracing, Geometry shader contour rendering, Gradient-based contour visualization, View-Dependent Contour Generation

## Overview
Object Contour NPR generates view-dependent lines that demarcate areas where an object's surface turns away from the viewer, effectively emphasizing physical boundaries [S1][S2, unofficial]. Unlike analytical methods that require explicit vector representations, this approach is pixel-based and relies on gradient estimation within the volume data [S1][S2, unofficial]. It is distinguished from sibling methods like Lasso Contour Rendering by its specific use of geometry shaders to trace line-strips from particle primitives, and from Silhouette Rendering by its reliance on local gradient magnitude and orientation rather than just visibility occlusion [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V(x,y,z)$, a viewing direction vector $\vec{v}$, and a user-defined distance threshold $d_{thresh}$ [S1][S2, unofficial].
*   **Output:** A set of line-strips rendered directly to the screen buffer [S1].

**Data Structures:**
1.  **Volume Data:** A 3D grid of scalar values.
2.  **Particle Set:** A collection of points sampled from the volume. Each particle $P_i$ has a position $\vec{p}_i$ and potentially a precomputed gradient $\nabla V_i$ [S1][S2, unofficial].
3.  **Geometry Shader State:** Configured to accept point primitives and output line strips [S1].

**Gradient Estimation:**
Conventional central difference estimation is insufficient due to its reliance on a narrow voxel neighborhood, leading to inaccuracies [S2, unofficial]. Instead, a 4D linear regression method is used to estimate the gradient $\nabla V$ at each particle position [S2, unofficial]. This method fits a plane to the scalar values in a local neighborhood to derive a more robust normal vector [S2, unofficial].

**Algorithm Steps:**
1.  **Particle Selection:** Iterate through the particle set. For each particle $P_i$ at position $\vec{p}_i$, calculate the gradient $\nabla V_i$ using 4D linear regression [S2, unofficial].
2.  **Contour Proximity Check:** Determine if the particle is near a contour. A contour is defined as a location where the surface turns away from the viewer. This is evaluated by the angle $\theta$ between the gradient vector $\nabla V_i$ and the viewing direction $\vec{v}$. Particles where the gradient magnitude $|\nabla V_i|$ is high and the orientation indicates a boundary are selected [S1][S2, unofficial]. Specifically, particles within a user-defined distance $d_{thresh}$ from the theoretical contour line are selected [S1].
3.  **Primitive Transformation:** Treat selected particles as point primitives [S1].
4.  **Geometry Shader Tracing:** Pass the point primitives to a geometry shader. The shader traces along the contour direction (derived from the gradient and viewing vector) to generate line-strips [S1]. The trace direction and constant-width measure are adopted from the VolumeFlies framework, which uses a threshold on a curvature-dependent measure $\tau$ [S1].
5.  **Rendering:** Render the generated line-strips directly to the screen buffer. They are not recorded to a texture buffer, unlike hatch strokes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_object_contours(volume, particles, view_direction, distance_threshold):
    """
    Renders object contours using geometry shaders.
    
    Args:
        volume: 3D array of scalar values.
        particles: List of particle objects with position and gradient.
        view_direction: Normalized 3D vector representing camera view.
        distance_threshold: Scalar distance for selecting particles near contours.
        
    Returns:
        None (renders directly to screen via GPU pipeline).
    """
    # Step 1 & 2: Filter particles based on gradient and proximity
    contour_particles = []
    for particle in particles:
        # Estimate gradient using 4D linear regression (see S2)
        # This replaces simple central differences
        gradient = estimate_gradient_4d_linear_regression(volume, particle.position)
        
        # Check if particle is near a contour
        # Contour condition: High gradient magnitude and specific orientation relative to view
        # The exact mathematical threshold for "turning away" is implied by the 
        # curvature-dependent measure tau from VolumeFlies [S1]
        is_near_contour = check_contour_proximity(
            particle.position, 
            gradient, 
            view_direction, 
            distance_threshold
        )
        
        if is_near_contour:
            contour_particles.append(particle)

    # Step 3: Send point primitives to GPU
    gpu_send_point_primitives(contour_particles)

    # Step 4: Geometry shader processes points into line strips
    # The shader uses the gradient and view direction to determine trace direction
    # and generates line segments of constant width based on tau [S1]
    gpu_execute_geometry_shader_trace()

    # Step 5: Render directly to screen
    gpu_render_to_screen()

def estimate_gradient_4d_linear_regression(volume, position):
    """
    Estimates gradient using 4D linear regression for better accuracy than central differences.
    Implements the method described in [S2, unofficial] referencing [NCKG00].
    """
    # Define local neighborhood around position
    neighborhood = get_local_voxel_neighborhood(volume, position)
    
    # Fit a linear model f(x,y,z,t) = ax + by + cz + d to the scalar values
    # where x,y,z are coordinates and t is the scalar value (or vice versa depending on formulation)
    # Solve for coefficients a, b, c which form the gradient vector
    coefficients = solve_linear_system(neighborhood)
    
    return coefficients[:3] # Return the gradient vector (a, b, c)

def check_contour_proximity(position, gradient, view_direction, threshold):
    """
    Determines if a particle is within the user-defined distance of a contour.
    Uses the curvature-dependent measure tau logic from VolumeFlies [S1].
    """
    # Calculate angle between gradient and view direction
    cos_theta = dot(normalize(gradient), normalize(view_direction))
    
    # Check gradient magnitude
    grad_mag = length(gradient)
    
    # Apply threshold logic
    # If grad_mag is high and orientation matches boundary criteria, return True
    # The exact formula for tau is not fully detailed in sources, but it depends on curvature [S1]
    # [G] Assuming a simple heuristic based on high gradient and back-facing orientation
    if grad_mag > 0.5 and cos_theta < -0.5: # Placeholder thresholds
        return True
    return False
```

**Parameters:**
*   **Distance Threshold ($d_{thresh}$):** User-defined distance from the contour for selecting particles [S1].
*   **Gradient Estimation Method:** 4D linear regression is required; central difference is explicitly discouraged [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Inaccuracy:** Using conventional central difference leads to significant errors because it only considers a narrow voxel neighborhood [S2, unofficial].
*   **Performance Cost:** Adding multiple particle sets for combined styles increases performance cost, potentially losing interactivity [S1].
*   **View Dependency:** Contours cannot be pre-rendered; they must be generated per frame [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of particles and volume size for gradient estimation. The geometry shader processing is efficient but depends on the number of selected contour particles [S1].
*   **Memory Complexity:** Requires storage for the volume data and the particle set. No additional texture buffers are needed for the contour lines themselves, as they are rendered directly [S1].
*   **Strengths:** Fast tuning because it does not depend on voxel intensity values or predefined classification functions [S2, unofficial]. It provides a rough estimation of physical boundaries effectively [S2, unofficial].
*   **Limitations:** Heavily reliant on accurate gradient reconstruction [S2, unofficial]. Performance degrades with multiple particle sets [S1].
*   **When to use:** When explicit analytical representations are unavailable and fast, view-dependent boundary visualization is needed [S1][S2, unofficial].
*   **When not to use:** When pre-computed, resolution-independent contours are required (use analytical methods) or when high precision in gradient estimation is not feasible [S2, unofficial].

## Background
Object Contour NPR was introduced by Csébfalvi et al. in 2001 [S2, unofficial]. It was adopted from the VolumeFlies framework [BVvW07], which presented constant-width contours using a threshold on a curvature-dependent measure $\tau$ [S1]. The method is part of the broader field of illustrative volume rendering, aiming to enhance shape perception through non-photorealistic techniques [S1][S2, unofficial].

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
