# Object Contour NPR

Also called: Csébfalvi's contour method, GPU-based contour tracing, Geometry shader contour rendering, Gradient-based contour visualization, View-Dependent Contour Generation

## Overview
Object Contour NPR generates view-dependent line primitives that demarcate areas where an object's surface turns away from the viewer, effectively conveying shape through boundary emphasis [S1]. Unlike surface-based NPR, this method operates on volumetric data by identifying regions of high gradient magnitude, estimating physical boundaries without relying on predefined classification functions or voxel intensity values [S2, unofficial]. It distinguishes itself from sibling methods like Lasso Contour Rendering by utilizing a geometry shader to trace line-strips directly to the screen from selected particles, rather than recording strokes to a texture buffer [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$, a viewing direction vector $\vec{v}$, and a user-defined distance threshold $d_{thresh}$ defining the neighborhood around the contour.
*   **Output:** A set of line-strips rendered directly to the screen, representing the object contours.

**Data Structures:**
1.  **Volume Grid:** A 3D array of scalar values.
2.  **Particle Set:** A collection of points $(x, y, z)$ distributed within the volume. Each particle stores its position and potentially precomputed gradient information.
3.  **Line-Strip Buffer:** A vertex buffer containing the generated line segments for the geometry shader.

**Gradient Estimation:**
Due to the method's heavy reliance on gradient accuracy, conventional central difference estimation is insufficient as it only considers a narrow voxel neighborhood [S2, unofficial]. Instead, a 4D linear regression method is used to compute the gradient $\nabla V$ at each particle position [S2, unofficial]. This provides a more robust estimation of the local surface orientation.

**Algorithm Steps:**
1.  **Particle Selection:** Iterate through the particle set. For each particle $p$ at position $\vec{p}$, compute the gradient $\nabla V(\vec{p})$. Select particles where the gradient magnitude $|\nabla V(\vec{p})|$ is high, indicating proximity to a physical boundary or contour [S2, unofficial]. Specifically, select particles within a user-defined distance $d_{thresh}$ from the implicit contour surface [S1].
2.  **View-Dependent Filtering:** Calculate the angle $\theta$ between the viewing direction $\vec{v}$ and the gradient vector $\nabla V(\vec{p})$. Contours are defined where the surface turns away from the viewer, which corresponds to specific angular relationships between $\vec{v}$ and the surface normal (derived from $\nabla V$) [S1].
3.  **Line-Strip Generation (Geometry Shader):** Treat the selected particles as point primitives. Pass these points to a geometry shader. The shader traces along the contour direction (derived from the curvature-dependent measure $\tau$ adopted from the VolumeFlies framework [S1]) to generate line-strips that resemble parts of the contour [S1].
4.  **Rendering:** Render the generated line-strips directly to the screen. Unlike hatch strokes which are recorded to a texture buffer, these line-strips are rendered immediately [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_object_contours(volume, particles, view_direction, distance_threshold):
    """
    Generates object contours for volume rendering.
    
    Args:
        volume: 3D array of scalar values.
        particles: List of 3D points (x, y, z).
        view_direction: Normalized 3D vector representing camera view.
        distance_threshold: Float, max distance from contour to select particle.
        
    Returns:
        list of line_strips: List of lists of 3D points forming line segments.
    """
    line_strips = []
    selected_particles = []
    
    # Step 1 & 2: Select particles based on gradient and distance
    for p in particles:
        # Compute gradient using 4D linear regression for accuracy [S2, unofficial]
        # Note: 4D linear regression implies using spatial coordinates (x,y,z) 
        # and time or another dimension, or a specific local fitting scheme.
        # Here we assume a function `compute_gradient_4d` exists.
        gradient = compute_gradient_4d(volume, p)
        grad_mag = magnitude(gradient)
        
        # Check if particle is near a contour boundary
        # The condition relies on high gradient magnitude and proximity
        if is_near_contour(p, gradient, view_direction, distance_threshold):
            selected_particles.append(p)
            
    # Step 3: Trace contours using geometry shader logic
    # In a CPU implementation, this would involve tracing along the contour line
    # For GPU, this is handled by the geometry shader transforming points to strips.
    # We simulate the output of the geometry shader here.
    
    # Group selected particles into connected components or traces
    # The trace direction is determined by the curvature-dependent measure tau [S1]
    traces = trace_contours(selected_particles, gradient_field=compute_gradient_field(volume))
    
    for trace in traces:
        line_strips.append(trace)
        
    return line_strips

def is_near_contour(p, gradient, view_direction, threshold):
    """
    Determines if a particle is near the object contour.
    Contours are where the surface turns away from the viewer.
    """
    # Normalize gradient to get surface normal
    normal = normalize(gradient)
    
    # Calculate angle between view direction and normal
    # If the surface turns away, the angle between view and normal is typically > 90 degrees
    # or the dot product is negative, depending on normal orientation.
    dot_product = dot(view_direction, normal)
    
    # Heuristic: Select if gradient is strong and orientation matches contour definition
    # Exact thresholding on dot_product depends on specific implementation of "turns away"
    if dot_product < 0 and magnitude(gradient) > SOME_HIGH_THRESHOLD:
        return True
    return False

def compute_gradient_4d(volume, point):
    """
    Computes gradient using 4D linear regression [S2, unofficial].
    This is a placeholder for the specific regression algorithm.
    """
    # Implementation detail: Fit a plane to local neighborhood in 4D space
    # (x, y, z, value) or similar.
    return [0, 0, 0] # Placeholder for actual regression result
```

**Parameters:**
*   **Distance Threshold ($d_{thresh}$):** User-defined distance from the contour for selecting particles [S1].
*   **Gradient Estimation Method:** Must use 4D linear regression; central difference is inaccurate [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Inaccuracy:** Using conventional central difference for gradient estimation leads to inaccurate results because it only considers a narrow voxel neighborhood [S2, unofficial].
*   **Performance Cost:** Adding multiple particle sets for different styles increases performance cost, potentially leading to loss of interactivity [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of particles for selection, plus the cost of gradient computation (4D regression is more expensive than central difference) and line tracing [S1][S2, unofficial].
*   **Memory Complexity:** Proportional to the number of particles and the size of the volume for gradient storage.
*   **Strengths:** Fast tuning because it has no direct dependence on voxel intensity values [S2, unofficial]. It provides a rough estimation of physical boundaries without predefined classification functions [S2, unofficial].
*   **Limitations:** Enormous reliance on accurate gradient reconstruction [S2, unofficial]. Performance degrades with multiple particle sets [S1].
*   **When to use:** When rapid visualization of object boundaries is needed without the overhead of isosurface extraction or complex classification [S2, unofficial].
*   **When not to use:** When high precision in gradient estimation is critical and computational resources for 4D regression are limited, or when static, view-independent contours are preferred.

## Background
Object Contour NPR was presented by Csébfalvi et al. in 2001 [S2, unofficial]. It was adopted from the VolumeFlies framework [BVvW07], which introduced constant-width contours using a threshold on a curvature-dependent measure $\tau$ [S1]. The method is part of the broader field of illustrative volume rendering, which seeks to enhance perceptual clarity through non-photorealistic edge representation [S2, S4, unofficial].

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
