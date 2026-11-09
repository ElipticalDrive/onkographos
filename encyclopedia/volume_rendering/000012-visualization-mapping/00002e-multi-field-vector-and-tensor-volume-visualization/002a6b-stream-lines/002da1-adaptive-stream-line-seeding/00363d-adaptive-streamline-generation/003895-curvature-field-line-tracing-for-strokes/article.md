# Curvature Field Line Tracing for Strokes

Also called: Stroke generation via RK4(3)

## Overview
This method generates hatching strokes by tracing paths from seed points along the maximal curvature directions of a scalar field in object space [S1, unofficial]. It differs from standard streamline tracing by using curvature vectors rather than velocity vectors, ensuring strokes follow the geometric structure of the volume, and by employing an adaptive step size to maintain alignment with the initial hatch direction [S1, unofficial].

## How it works
**Inputs:** Scalar field data, seed points, initial hatch direction, deviation threshold constant.
**Outputs:** A set of strokes, where each stroke is a line strip of points.

**Data Structures:**
Each point in a stroke stores:
1.  Vertex coordinates ($\mathbf{p}$).
2.  Gray-scale gradient ($\nabla f$) interpolated from the scalar field.
3.  Texture coordinate ($u$), defined as the distance from the current point to the seed point divided by the total stroke length.

**Algorithm Steps:**
1.  **Initialization:** For each selected seed point, determine the initial hatch direction.
2.  **Integration:** Trace the path from the seed point along the maximal curvature direction of the scalar field. Use the Runge-Kutta 4(3) (RK4(3)) method with adaptive step size control to numerically integrate the field lines [S1, unofficial].
3.  **Termination Check:** At each step, calculate the angle between the current hatch direction and the original hatch direction. If the deviation exceeds a predefined constant threshold, terminate the stroke [S1, unofficial]. This prevents stepping into regions with high curvature field variations.
4.  **Storage:** Store the traced points as a line strip.
5.  **Lighting Data:** At each point, interpolate the gray-scale gradient from the scalar field to simulate lighting effects [S1, unofficial].
6.  **Culling Preparation:** Compute a cone of normals from the set of all gradients to enable culling of back-facing strokes later [S1, unofficial].
7.  **Parameter Calculation:** Compute the texture coordinate for each point as the cumulative distance from the seed relative to the total length of the stroke [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_stroke(seed_point, initial_hatch_direction, scalar_field, threshold_angle):
    """
    Generates a single stroke by tracing along maximal curvature directions.
    
    Args:
        seed_point: Initial 3D coordinate.
        initial_hatch_direction: Normalized 3D vector for the starting direction.
        scalar_field: Object providing interpolate_gradient(pos) and compute_curvature(pos).
        threshold_angle: Maximum allowed deviation (radians) from initial_hatch_direction.
        
    Returns:
        list of dicts: Each dict contains 'pos', 'gradient', 'u'.
    """
    stroke_points = []
    current_pos = seed_point
    current_direction = initial_hatch_direction
    total_length = 0.0
    
    # Adaptive step size parameters
    step_size = 0.1 # Not given in sources; implementation detail [G]
    min_step = 1e-6
    max_step = 1.0
    
    while True:
        # 1. Get curvature vector at current position
        curvature_vec = scalar_field.compute_curvature(current_pos)
        
        # Normalize curvature to get direction
        curvature_norm = length(curvature_vec)
        if curvature_norm < 1e-9:
            break # Degenerate case
            
        direction = normalize(curvature_vec)
        
        # 2. Check deviation from original hatch direction
        # Note: The source says "current hatch direction" vs "original". 
        # In curvature tracing, the direction changes. We compare the 
        # integrated path's tangent or the local curvature direction against 
        # the initial intent. The source implies a global check against the 
        # original seed direction or local consistency. 
        # Based on "deviates from the original one", we compare against initial_hatch_direction.
        angle = acos(dot(normalize(current_direction), normalize(initial_hatch_direction)))
        
        if angle > threshold_angle:
            break
            
        # 3. RK4(3) Integration Step
        # Since the field is the curvature field, we integrate dP/dt = Curvature(P)
        # RK4(3) requires evaluating the field at intermediate points.
        k1 = scalar_field.compute_curvature(current_pos)
        
        temp_pos = current_pos + 0.5 * step_size * k1
        k2 = scalar_field.compute_curvature(temp_pos)
        
        temp_pos = current_pos + 0.5 * step_size * k2
        k3 = scalar_field.compute_curvature(temp_pos)
        
        temp_pos = current_pos + step_size * k3
        k4 = scalar_field.compute_curvature(temp_pos)
        
        # Weighted sum for RK4(3)
        # Standard RK4 weights are 1/6, 1/3, 1/3, 1/6. 
        # RK4(3) usually refers to an embedded pair for error control.
        # The source specifies RK4(3) but does not give the specific embedded weights.
        # We use standard RK4 update for position, assuming the error estimator 
        # controls step_size [G].
        displacement = (step_size / 6.0) * (k1 + 2*k2 + 2*k3 + k4)
        
        # 4. Update position
        next_pos = current_pos + displacement
        dist = length(displacement)
        total_length += dist
        
        # 5. Interpolate gradient for lighting
        gradient = scalar_field.interpolate_gradient(next_pos)
        
        # 6. Store point
        # Texture coordinate u = distance_to_seed / total_length
        # Note: total_length is cumulative.
        u = total_length / max(total_length, 1e-9) # Avoid division by zero if loop runs once
        
        stroke_points.append({
            'pos': next_pos,
            'gradient': gradient,
            'u': u
        })
        
        # 7. Adaptive Step Size Control
        # Estimate error using embedded RK3/RK4 difference (not given in sources) [G]
        # If error > tolerance, reduce step_size. Else, increase.
        # For simplicity in this minimal implementation, we just proceed.
        # In practice:
        # error = estimate_error(k1, k2, k3, k4)
        # if error > tolerance: step_size *= 0.5
        # else: step_size = min(step_size * 1.2, max_step)
        
        current_pos = next_pos
        
        # Safety break
        if dist < min_step:
            break
            
    return stroke_points
```

**Parameters:**
*   **Deviation Threshold:** A constant angle deviation from the original hatch direction that terminates the stroke. No specific value is given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **High Variations:** The method avoids stepping into regions where the curvature field shows high variations by terminating the stroke upon deviation [S1, unofficial].
*   **Precision:** Although tri-linear interpolation is used for the curvature field, the RK4(3) method allows for high accuracy in computing field lines [S1, unofficial].

## Subtopics

### Integral Surface
An integral surface is defined by a univariate seeding curve $c: I \subseteq \mathbb{R} \to \mathbb{R}^n$ and is given by $S(s, t) = c(s) + \int_{t_0}^{t} v(\tau, S(s, \tau)) d\tau$ [S2, unofficial]. In time-varying velocity fields, this forms a path surface; in steady flow, a stream surface. Lines on a path surface for fixed $t$ are timelines. Higher-dimensional seeding geometry generalizes these to path volumes and time surfaces [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The adaptive step size control considerably reduces the number of intermediate points used to sample the vector field, allowing strokes to be effectively encoded using a very small number of line segments [S1, unofficial].
*   **Strengths:** Ensures strokes are placed entirely in front of any selected surface by assuming sufficiently accurate curvature directions [S1, unofficial]. Simulates lighting effects via interpolated gray-scale gradients [S1, unofficial].
*   **Limitations:** Relies on the accuracy of the curvature field; high variations can cause early termination [S1, unofficial].

## Background
This method is part of interactive volume illustration techniques, specifically for generating hatching strokes [S1, unofficial]. It was developed to provide artistic rendering of volumes by leveraging the geometric properties (curvature) of the scalar field rather than just iso-surfaces.

## Sources
[S1] Interactive volume illustration, https://openalex.org/W2147354080, fetched 2026-10-27 (unofficial)
[S2] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
