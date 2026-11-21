# Adaptive Convergence Threshold

Also called: Camera-aware SDF Threshold, Convergence Criteria for Sphere Tracing, Pixel-intersection based epsilon

## Overview
Adaptive Convergence Threshold is a stopping condition for distance-field ray marching that determines when the ray has sufficiently converged to the surface to halt further computation [S1]. It solves the trade-off between rendering accuracy and computational cost by dynamically calculating a threshold $\epsilon$ based on camera parameters, ensuring that the ray front ends of neighboring pixels do not intersect [S1]. Unlike fixed thresholds, this method scales with the scene geometry and camera setup to prevent both dilation (loss of detail) and erosion (incomplete geometry) [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $d_{min}$: Minimum distance from the camera to the object [S1].
    *   $S$: Sensor size (width) [S1].
    *   $f$: Focal length [S1].
    *   $R$: Image resolution (width) [S1].
    *   $\theta$: Angle of the ray relative to the optical axis [S1].
    *   $SDF(p)$: The signed distance function value at the current ray position $p$ [S1].
*   **Outputs:**
    *   $\epsilon$: The convergence threshold [S1].
    *   Boolean: Whether to continue marching ($|SDF(p)| \ge \epsilon$) or stop ($|SDF(p)| < \epsilon$) [S1].

**Data Structures**
No specific data structures are required beyond standard scalar variables for camera parameters and the current SDF evaluation [G].

**Steps**
1.  Calculate the adaptive convergence threshold $\epsilon$ using the camera intrinsics and the ray's angle $\theta$ [S1].
2.  Evaluate the signed distance function $SDF(p)$ at the current ray position $p$ [S1].
3.  Compare the absolute SDF value to the threshold:
    *   If $|SDF(p)| < \epsilon$, stop marching [S1].
    *   Otherwise, continue marching [S1].

**Formula**
The threshold $\epsilon$ is derived from the condition that balls centered at the ray front ends of neighboring pixels must not intersect [S1]. The formula is:

$$ \epsilon = \frac{d_{min} \cdot S \cdot \cos^2(\theta)}{2 \cdot f \cdot R} $$

Where:
*   $d_{min}$ is the minimum object distance [S1].
*   $S$ is the sensor size [S1].
*   $f$ is the focal length [S1].
*   $R$ is the resolution [S1].
*   $\theta$ is the angle of the ray [S1].

**Parameters**
*   **Typical Values:** For a setup with $f=60$mm, $d_{min}=10$cm, $S=32$mm, and $R=512$, the resulting $\epsilon \approx 0.5 \times 10^{-4}$m (0.05mm) [S1].
*   **Practical Threshold:** In practice, a fixed threshold of $5 \times 10^{-5}$ is often used for experiments, though the adaptive calculation provides a theoretical bound [S1].

**Pitfalls**
*   **Too Large:** A threshold that is too large causes dilation, leading to the loss of boundary details and large depth errors due to bounded rendering noise [S1].
*   **Too Small:** A threshold that is too small significantly increases rendering time [S1]. If a maximum number of tracing steps is enforced, pixels may fail to converge, resulting in erosion (background misclassification) [S1].
*   **Angle Dependence:** The threshold varies with $\theta$ [S1]. Rays at larger angles (peripheral pixels) have a smaller $\epsilon$, requiring more precision [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_adaptive_threshold(d_min, S, f, R, theta):
    """
    Calculate the adaptive convergence threshold epsilon.
    
    Args:
        d_min (float): Minimum distance to object.
        S (float): Sensor size.
        f (float): Focal length.
        R (float): Resolution.
        theta (float): Ray angle in radians.
        
    Returns:
        float: The convergence threshold epsilon.
    """
    cos_theta = math.cos(theta)
    epsilon = (d_min * S * cos_theta * cos_theta) / (2 * f * R)
    return epsilon

def ray_marching_step(p, SDF_func, epsilon):
    """
    Perform one step of ray marching with adaptive convergence check.
    
    Args:
        p (Vector3): Current ray position.
        SDF_func (function): Signed Distance Function evaluator.
        epsilon (float): Convergence threshold.
        
    Returns:
        bool: True if marching should continue, False if converged.
    """
    dist = SDF_func(p)
    if abs(dist) < epsilon:
        return False  # Converged
    return True       # Continue
```

## Operating characteristics
*   **Time Complexity:** Rendering time scales with the number of queries needed to reach the threshold, increasing significantly if the threshold is set too small [S1].
*   **Memory Complexity:** $O(1)$ additional memory for the threshold calculation [G].
*   **Strengths:** Prevents dilation and erosion by aligning the convergence criterion with the pixel resolution [S1]. Provides a theoretical bound on depth error [S1].
*   **Limitations:** Requires knowledge of camera intrinsics ($f, S, R$) and an estimate of $d_{min}$ [S1]. The calculation of $\theta$ requires ray direction knowledge [S1].
*   **When to Use:** When rendering with differentiable sphere tracing where depth accuracy and silhouette precision are critical, and camera parameters are known [S1].
*   **When Not to Use:** When camera parameters are unknown or dynamic in a way that makes $d_{min}$ estimation unreliable [S1].

## Background
The method was introduced in the context of differentiable rendering of implicit signed distance functions (SDFs) [S1]. It addresses the issue of slow convergence near the surface where the SDF value approaches zero [S1]. The approach is based on the geometric principle that the threshold should be small enough to distinguish between neighboring pixels but large enough to avoid excessive computation [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
