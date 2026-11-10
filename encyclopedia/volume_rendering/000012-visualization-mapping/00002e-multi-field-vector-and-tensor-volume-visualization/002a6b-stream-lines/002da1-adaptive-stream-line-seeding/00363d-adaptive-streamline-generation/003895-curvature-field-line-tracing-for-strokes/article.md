# Curvature Field Line Tracing for Strokes
Also called: Stroke generation via RK4(3)

## Overview
Curvature Field Line Tracing for Strokes is a method for generating hatching strokes in volume illustration by tracing paths from seed points along maximal curvature directions in object space [S1, unofficial]. It solves the problem of creating visually coherent, front-facing hatching lines that follow the shape of the underlying scalar field's geometry. It is distinguished from standard streamline tracing by integrating along curvature gradients rather than velocity vectors, and by using a deviation-based termination criterion to ensure strokes remain locally planar and front-facing [S1, unofficial].

## How it works
**Inputs:**
- A scalar field $f(\mathbf{x})$ defined on a 3D grid.
- A set of seed points $\mathbf{p}_0$ located on or near an iso-surface of interest.
- A deviation threshold angle $\theta_{max}$ (constant) [S1, unofficial].

**Outputs:**
- A set of strokes, where each stroke is a line strip (sequence of points).

**Data Structures:**
Each point in a stroke stores:
1. Vertex coordinates $\mathbf{p} \in \mathbb{R}^3$.
2. Gray-scale gradient $\nabla f(\mathbf{p})$ interpolated from the scalar field.
3. Texture coordinate $t$, defined as the cumulative distance from the seed point divided by the total stroke length [S1, unofficial].

**Algorithm Steps:**

1. **Initialize:** For each seed point $\mathbf{p}_0$, compute the initial maximal curvature direction $\mathbf{d}_0$. This direction is derived from the curvature tensor or principal curvatures of the scalar field at $\mathbf{p}_0$ [S1, unofficial]. Let the initial hatch direction be $\mathbf{h}_0 = \mathbf{d}_0$.

2. **Integration Loop:** While the stroke is not terminated:
   a. **Compute Derivatives:** Evaluate the curvature field $\mathbf{C}(\mathbf{x})$ and its spatial derivatives at the current position $\mathbf{p}$ to determine the direction of maximal curvature. Tri-linear interpolation is used for the field values [S1, unofficial].
   b. **Adaptive RK4(3) Step:** Use a Runge-Kutta 4(3) method with adaptive step size control to integrate the differential equation $\frac{d\mathbf{p}}{ds} = \mathbf{C}(\mathbf{p})$. The adaptive step size reduces the number of intermediate points by increasing step size in smooth regions and decreasing it in high-variation regions [S1, unofficial].
   c. **Update Position:** Move to the new point $\mathbf{p}_{new}$.
   d. **Termination Check:** Calculate the angle $\alpha$ between the current hatch direction $\mathbf{h}_{current}$ and the original hatch direction $\mathbf{h}_0$. If $\alpha > \theta_{max}$, terminate the stroke [S1, unofficial]. This prevents stepping into regions where the curvature field varies highly or flips direction significantly [S1, unofficial].
   e. **Store Point:** Add $\mathbf{p}_{new}$ to the stroke's point list.
   f. **Interpolate Gradient:** Compute $\nabla f(\mathbf{p}_{new})$ for lighting simulation [S1, unofficial].
   g. **Update Texture Coordinate:** Update $t$ based on the distance traveled from the seed [S1, unofficial].

3. **Post-Processing:**
   a. **Cone of Normals:** From the set of all gradients $\nabla f$ along the stroke, derive a cone of normals. This is used to cull back-facing strokes during rendering, ensuring only strokes facing the viewer are displayed [S1, unofficial].
   b. **Lighting:** Use the interpolated gray-scale gradients to simulate lighting effects on the strokes [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_curvature_stroke(seed_point, scalar_field, theta_max):
    """
    Trace a single hatching stroke along maximal curvature directions.
    
    Args:
        seed_point: Initial 3D coordinate.
        scalar_field: Object providing tri-linear interpolation of scalar values and gradients.
        theta_max: Maximum allowed deviation angle from the initial hatch direction (radians).
        
    Returns:
        list of dict: Each dict contains 'pos' (vec3), 'gradient' (vec3), 't' (float).
    """
    # 1. Initialize
    current_pos = seed_point
    # Compute initial maximal curvature direction
    initial_hatch_dir = compute_maximal_curvature_direction(scalar_field, current_pos)
    hatch_dir = initial_hatch_dir
    
    stroke_points = []
    current_pos = seed_point
    
    # Add initial point
    stroke_points.append({
        'pos': current_pos,
        'gradient': scalar_field.interpolate_gradient(current_pos),
        't': 0.0
    })
    
    total_length = 0.0
    step_size = initial_step_size # [G] Initial step size not specified in sources
    
    while True:
        # 2. Adaptive RK4(3) Integration
        # Evaluate curvature field derivatives at current_pos
        # The field to integrate is the maximal curvature direction field
        curvature_field = get_curvature_direction_field(scalar_field)
        
        # Perform one RK4(3) step with adaptive control
        # This returns the new position and the actual step size taken
        next_pos, actual_step = adaptive_rk43_step(
            curvature_field, 
            current_pos, 
            step_size, 
            tolerance=1e-4 # [G] Tolerance not specified, assumed small for accuracy
        )
        
        # 3. Termination Check
        # Calculate angle between current hatch direction (initial) and the direction to next point
        # Note: The source says "deviates from the original one", implying comparison with initial hatch_dir
        direction_to_next = normalize(next_pos - current_pos)
        angle = acos(dot(normalize(hatch_dir), direction_to_next))
        
        if angle > theta_max:
            break
            
        # 4. Update State
        current_pos = next_pos
        total_length += actual_step
        
        # 5. Store Point
        stroke_points.append({
            'pos': current_pos,
            'gradient': scalar_field.interpolate_gradient(current_pos),
            't': total_length # [G] Normalization by total length happens after or during render
        })
        
        # 6. Adaptive Step Size Adjustment
        # RK4(3) typically adjusts step size based on error estimate
        # This is handled internally by adaptive_rk43_step
        
    # Normalize texture coordinates
    if total_length > 0:
        for point in stroke_points:
            point['t'] /= total_length
            
    return stroke_points

def compute_maximal_curvature_direction(scalar_field, pos):
    """
    Compute the direction of maximal curvature at a point.
    This involves computing the Hessian or curvature tensor of the scalar field.
    """
    # [G] Specific formula for maximal curvature direction from scalar field
    # is not detailed in the source, but generally involves eigen-decomposition
    # of the Hessian matrix or curvature tensor.
    hessian = scalar_field.compute_hessian(pos)
    eigenvalues, eigenvectors = hessian.eigendecompose()
    # Return eigenvector corresponding to the largest absolute eigenvalue
    return eigenvectors[:, argmax(abs(eigenvalues))]

def get_curvature_direction_field(scalar_field):
    """
    Returns a function that computes the maximal curvature direction at any point.
    """
    def field_func(pos):
        return compute_maximal_curvature_direction(scalar_field, pos)
    return field_func

def adaptive_rk43_step(field_func, pos, h, tolerance):
    """
    Perform one step of RK4(3) with adaptive step size.
    Returns (next_pos, actual_step_size).
    """
    # [G] Standard RK4(3) implementation details omitted for brevity
    # The key is that it estimates error and adjusts h accordingly.
    # The source emphasizes that this reduces the number of points
    # while maintaining high accuracy despite tri-linear interpolation.
    pass
```

**Edge Cases and Pitfalls:**
- **High Variation Regions:** The deviation threshold $\theta_{max}$ prevents the stroke from entering regions where the curvature field varies highly, which could cause erratic behavior [S1, unofficial].
- **Tri-linear Interpolation:** Although the curvature field is only tri-linearly interpolated, the RK4(3) method ensures high accuracy in field line computation [S1, unofficial].
- **Back-facing Strokes:** The cone of normals derived from gradients is essential for culling back-facing strokes, ensuring visual correctness [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Depends on the number of seed points and the length of the strokes. The adaptive step size reduces the number of intermediate points, improving performance in smooth regions [S1, unofficial].
- **Memory Complexity:** Proportional to the number of points in all strokes. Each point stores position, gradient, and texture coordinate [S1, unofficial].
- **Strengths:** High accuracy in field line tracing despite low-order interpolation; effective culling of back-facing strokes; visually coherent hatching that follows object shape [S1, unofficial].
- **Limitations:** Requires computation of curvature directions, which may be more expensive than simple gradient-based streamlines; termination criterion may truncate strokes prematurely if $\theta_{max}$ is too small [S1, unofficial].
- **When to Use:** When visualizing scalar fields with hatching strokes that need to follow the geometric shape of the data, particularly in interactive volume illustration [S1, unofficial].

## Background
The method was introduced in the context of interactive volume illustration, where hatching strokes are used to convey shape and depth [S1, unofficial]. It builds on the idea of using curvature information to guide the placement of strokes, ensuring they align with the principal directions of the surface geometry. The use of RK4(3) with adaptive step size is a numerical technique chosen for its balance of accuracy and efficiency [S1, unofficial].

## Sources
[S1] Interactive volume illustration, https://openalex.org/W2147354080, fetched 2026-10-27 (unofficial)
