# Adaptive Convergence Threshold

Also called: Camera-aware SDF Threshold, Convergence Criteria for Sphere Tracing, Pixel-intersection based epsilon

## Overview
Adaptive Convergence Threshold is a stopping condition for distance-field ray marching that determines when a ray has sufficiently converged to a surface. It solves the problem of balancing rendering accuracy against computational cost by defining a threshold $\epsilon$ based on camera parameters, ensuring that the ray front ends of neighboring pixels remain geometrically separable. It differs from fixed thresholds by adapting to the scene scale and camera intrinsics to prevent both dilation (loss of detail) and erosion (incomplete geometry) [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Camera intrinsics (focal length $f$, sensor size $S$, resolution $R$), scene scale (minimum object distance $d_{min}$), viewing angle $\theta$, and the current signed distance function (SDF) value $f(\mathbf{p})$ at the ray position $\mathbf{p}$.
- **Outputs:** A boolean convergence flag and the computed threshold $\epsilon$.

**Data Structures:**
No complex data structures are required. The method relies on scalar parameters derived from camera calibration and scene bounds.

**Steps:**
1.  **Calculate Threshold:** Compute the adaptive threshold $\epsilon$ using the camera parameters and scene distance. This ensures that the "ball" of uncertainty around the ray front end does not overlap with neighboring pixels' balls [S1].
2.  **Evaluate Convergence:** At each ray marching step, check if the absolute SDF value is below the threshold:
    $$ |f(\mathbf{p})| < \epsilon $$
3.  **Terminate:** If the condition is met, stop marching. The current position is considered the surface intersection for rendering purposes [S1].

**Formula:**
The threshold $\epsilon$ is derived from similar triangles, representing the radius such that balls centered at the ray front ends of neighboring pixels do not intersect [S1]:

$$ \epsilon = \frac{d_{min} \cdot S \cdot \cos^2(\theta)}{2 \cdot f \cdot R} $$

Where:
- $d_{min}$: Minimum distance from the camera to the object of interest.
- $S$: Sensor size (width or height, consistent with $R$).
- $f$: Focal length.
- $R$: Image resolution (width or height).
- $\theta$: The angle between the ray direction and the camera optical axis.

**Parameters:**
- Typical values for $\epsilon$ calculation: $f=60$mm, $d_{min}=10$cm, $S=32$mm, $R=512$. This yields $\epsilon \approx 0.5 \times 10^{-4}$m (0.05mm) [S1].
- In practice, a fixed threshold of $5 \times 10^{-5}$ is often used for experiments with similar setups [S1].

**Pitfalls:**
- **Too Large:** Causes dilation, where the rendered shape is larger than the true geometry, losing boundary details. It also bounds rendering noise, leading to large depth errors [S1].
- **Too Small:** Causes erosion, where geometry is incomplete because pixels fail to converge within the maximum step count, misclassifying foreground as background. It also significantly increases rendering time due to excessive queries [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_adaptive_threshold(d_min, sensor_size, focal_length, resolution, angle_theta):
    """
    Computes the adaptive convergence threshold epsilon.
    
    Args:
        d_min (float): Minimum distance to object (meters).
        sensor_size (float): Sensor size (meters).
        focal_length (float): Focal length (meters).
        resolution (int): Image resolution (pixels).
        angle_theta (float): Angle from optical axis (radians).
        
    Returns:
        float: The convergence threshold epsilon.
    """
    cos_theta = math.cos(angle_theta)
    epsilon = (d_min * sensor_size * cos_theta**2) / (2.0 * focal_length * resolution)
    return epsilon

def check_convergence(sdf_value, epsilon):
    """
    Checks if the ray has converged to the surface.
    
    Args:
        sdf_value (float): The signed distance function value at current position.
        epsilon (float): The adaptive threshold.
        
    Returns:
        bool: True if converged, False otherwise.
    """
    return abs(sdf_value) < epsilon
```

## Subtopics

### System Stability Check
A variant used in particle-based surface extraction rather than ray marching. It defines system equilibrium as energy minimization where the energy gradient is zero. To avoid complex gradient calculations, it uses the average displacement of particles to characterize the gradient. The system is considered to have approached equilibrium if the average displacement $\Delta(l)$ is less than a threshold $\epsilon$ for $M$ consecutive iterations. The threshold is proportional to the particle repulsion radius $h_{cut\_off}$: $\epsilon = \beta \cdot h_{cut\_off}$, where $\beta$ is a coefficient (e.g., 0.01) [S2, unofficial]. The average displacement is calculated as $\Delta(t) = \frac{1}{N} \sum_{i=1}^{N} |p_i(t) - p_i(t-1)|$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces the number of SDF queries per ray by stopping early when close to the surface, improving performance compared to fixed small thresholds [S1].
- **Memory Complexity:** Negligible; requires only scalar storage for camera parameters.
- **Strengths:** Prevents both dilation and erosion by tying the threshold to the physical resolution of the rendering pipeline. It ensures that the rendered details are supported by the 2D image resolution [S1].
- **Limitations:** Requires knowledge of camera intrinsics and scene scale ($d_{min}$). If $d_{min}$ is inaccurate, the threshold may be inappropriate. It assumes a pinhole camera model for the derivation [S1].
- **When to Use:** When rendering implicit surfaces with differentiable sphere tracing where depth accuracy and boundary detail preservation are critical, and camera parameters are known [S1].

## Background
The method was introduced in the context of differentiable rendering of deep implicit signed distance functions (SDFs). It addresses the issue where ray movement becomes extremely slow near the surface because the SDF value $f$ approaches zero. The concept was proposed to ensure that the convergence criteria do not confuse one pixel with its neighbors, thereby maintaining details supported by the 2D rendering resolution [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] Surface Extraction for Industrial CT Based On Adaptive Particle System, https://doi.org/10.21203/rs.3.rs-9502773/v1, fetched 2026-10-26 (unofficial)
