# Camera Model with Radial/Tangential Distortion

Also called: nerfies Camera

## Overview
This camera model defines the geometric relationship between 2D pixel coordinates and 3D world rays, incorporating standard pinhole camera intrinsics (focal length, principal point, skew, aspect ratio) and lens distortion (radial and tangential) [S1, unofficial]. It is primarily used in neural rendering pipelines, such as Nerfies, to accurately map pixel locations to world-space directions for ray generation [S1, unofficial]. It distinguishes itself from simple pinhole models by including an iterative undistortion step to recover ideal normalized coordinates from distorted pixel measurements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Pixel coordinates (shape $[..., 2]$), camera intrinsic parameters, camera extrinsic parameters (orientation, position).
*   **Outputs:** Normalized ray directions in world coordinates (shape $[..., 3]$).

**Data Structures:**
The `Camera` object stores the following fields [S1, unofficial]:
*   `orientation`: $3 \times 3$ matrix (world-to-camera rotation).
*   `position`: 3-vector (camera center in world coordinates).
*   `focal_length`: float.
*   `principal_point`: 2-vector $[u_0, v_0]$.
*   `skew`: float.
*   `pixel_aspect_ratio`: float.
*   `radial_distortion`: 3-vector $[k_1, k_2, k_3]$.
*   `tangential_distortion`: 2-vector $[p_1, p_2]$.
*   `image_size`: 2-vector $[width, height]$ (uint32).
*   `dtype`: data type (e.g., float32).

**Formulas:**
1.  **Pixel to Local Undistorted Coordinates:**
    The mapping from ideal normalized coordinates $(x, y)$ to distorted pixel coordinates $(u, v)$ is [S1, unofficial]:
    $$ u = f \cdot x + s \cdot y + u_0 $$
    $$ v = f \cdot \text{aspect} \cdot y + v_0 $$
    Where $f$ is focal length, $s$ is skew, and $\text{aspect}$ is pixel aspect ratio.

2.  **Radial Distortion Factor:**
    $$ r^2 = x^2 + y^2 $$
    $$ d(r) = 1 + r^2 (k_1 + r^2 (k_2 + k_3 r^2)) $$

3.  **Tangential and Radial Distortion Application:**
    The distorted coordinates $(x_d, y_d)$ are related to ideal coordinates $(x, y)$ by [S1, unofficial]:
    $$ x_d = x \cdot d(r) + 2 p_1 x y + p_2 (r^2 + 2 x^2) $$
    $$ y_d = y \cdot d(r) + 2 p_2 x y + p_1 (r^2 + 2 y^2) $$

4.  **World Ray Direction:**
    $$ \mathbf{d}_{world} = \frac{\text{Orientation}^T \cdot \mathbf{d}_{local}}{\|\mathbf{d}_{local}\|} $$

**Algorithm Steps:**
1.  **Validation:** Ensure pixel coordinates have a last dimension of 2 and match the camera's dtype [S1, unofficial].
2.  **Reshape:** Flatten pixel coordinates to $(-1, 2)$ for batch processing [S1, unofficial].
3.  **Normalize to Principal Point:**
    Calculate intermediate $y$ and $x$ values [S1, unofficial]:
    $$ y = \frac{v - v_0}{f \cdot \text{aspect}} $$
    $$ x = \frac{u - u_0 - s \cdot y}{f} $$
4.  **Undistortion:** If radial or tangential distortion parameters are non-zero, apply `_radial_and_tangential_undistort` to $(x, y)$ using Newton's method to solve for ideal $(x, y)$ given distorted $(x_d, y_d)$ [S1, unofficial].
5.  **Local Ray Direction:** Stack $[x, y, 1]$ and normalize by L2 norm [S1, unofficial].
6.  **World Transformation:** Multiply local directions by the transpose of the orientation matrix [S1, unofficial].
7.  **Final Normalization:** Normalize the resulting world directions by L2 norm [S1, unofficial].
8.  **Reshape:** Restore the original batch shape plus $(3,)$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def _compute_residual_and_jacobian(x, y, xd, yd, k1, k2, k3, p1, p2):
    """Computes residuals and Jacobian for undistortion."""
    r = x * x + y * y
    d = 1.0 + r * (k1 + r * (k2 + k3 * r))
    
    # Distorted projection equations
    fx = d * x + 2 * p1 * x * y + p2 * (r + 2 * x * x) - xd
    fy = d * y + 2 * p2 * x * y + p1 * (r + 2 * y * y) - yd
    
    # Derivatives of d
    d_r = k1 + r * (2.0 * k2 + 3.0 * k3 * r)
    d_x = 2.0 * x * d_r
    d_y = 2.0 * y * d_r
    
    # Jacobian entries
    fx_x = d + d_x * x + 2 * p1 * y + 4 * p2 * x
    fx_y = d_x * y + 2 * p1 * x + 2 * p2 * y
    fy_x = d_x * y + 2 * p2 * y + 2 * p1 * x
    fy_y = d + d_y * y + 2 * p2 * x + 4 * p1 * y
    
    return fx, fy, fx_x, fx_y, fy_x, fy_y

def _radial_and_tangential_undistort(xd, yd, k1=0.0, k2=0.0, k3=0.0, p1=0.0, p2=0.0, eps=1e-9, max_iterations=10):
    """Undistorts pixel coordinates using Newton's method."""
    x = xd.copy()
    y = yd.copy()
    
    for _ in range(max_iterations):
        fx, fy, fx_x, fx_y, fy_x, fy_y = _compute_residual_and_jacobian(
            x, y, xd, yd, k1, k2, k3, p1, p2)
        
        denominator = fy_x * fx_y - fx_x * fy_y
        x_numerator = fx * fy_y - fy * fx_y
        y_numerator = fy * fx_x - fx * fy_x
        
        # Avoid division by zero
        step_x = np.where(np.abs(denominator) > eps, x_numerator / denominator, 0.0)
        step_y = np.where(np.abs(denominator) > eps, y_numerator / denominator, 0.0)
        
        x = x + step_x
        y = y + step_y
        
    return x, y

def pixels_to_rays(pixels, camera):
    """
    Convert pixel coordinates to world-space ray directions.
    
    Args:
        pixels: np.ndarray of shape (..., 2)
        camera: Camera object with attributes orientation, position, focal_length, 
                principal_point, skew, pixel_aspect_ratio, radial_distortion, 
                tangential_distortion.
                
    Returns:
        np.ndarray of shape (..., 3) containing normalized ray directions.
    """
    # 1. Validation
    if pixels.shape[-1] != 2:
        raise ValueError("Pixels must have shape (..., 2)")
    
    # 2. Reshape
    original_shape = pixels.shape
    pixels_flat = pixels.reshape(-1, 2)
    
    # 3. Normalize to principal point
    # y = (v - v0) / (f * aspect)
    # x = (u - u0 - s * y) / f
    u = pixels_flat[:, 0]
    v = pixels_flat[:, 1]
    
    f = camera.focal_length
    aspect = camera.pixel_aspect_ratio
    u0, v0 = camera.principal_point
    s = camera.skew
    
    y = (v - v0) / (f * aspect)
    x = (u - u0 - s * y) / f
    
    # 4. Undistortion
    if camera.has_radial_distortion or camera.has_tangential_distortion:
        k1, k2, k3 = camera.radial_distortion
        p1, p2 = camera.tangential_distortion
        x, y = _radial_and_tangential_undistort(x, y, k1, k2, k3, p1, p2)
        
    # 5. Local Ray Direction
    dirs = np.stack([x, y, np.ones_like(x)], axis=-1)
    dirs = dirs / np.linalg.norm(dirs, axis=-1, keepdims=True)
    
    # 6. World Transformation
    # rays_dir = orientation.T @ local_dirs
    # Note: orientation is 3x3. local_dirs is (N, 3).
    # We need to rotate local vectors to world space.
    # If orientation is world-to-camera, then camera-to-world is orientation.T.
    # Matrix multiplication: (N, 3) @ (3, 3).T -> (N, 3)
    # Or: (3, 3).T @ (N, 3).T -> (3, N).T -> (N, 3)
    # Using matmul with newaxis for broadcasting if needed, but here simple dot works.
    rays_dir = np.dot(dirs, camera.orientation.T)
    
    # 7. Final Normalization
    rays_dir = rays_dir / np.linalg.norm(rays_dir, axis=-1, keepdims=True)
    
    # 8. Reshape
    return rays_dir.reshape(original_shape[:-1] + (3,))
```

**Edge Cases and Pitfalls:**
*   **Zero Denominator:** In undistortion, if the Jacobian determinant is near zero, the step is clamped to zero to prevent NaNs [S1, unofficial].
*   **Degenerate Look-At:** If the look-at vector and up vector are parallel, the right vector calculation fails (norm < eps) [S1, unofficial].
*   **Cropping:** If cropping results in non-positive resolution, an error is raised [S1, unofficial].

## Subtopics

### Pixel Scale
A parameterization variant where pixel-valued parameters (focal length, principal point) are scaled by the initial camera focal length $f$ [S2]. This encodes camera residual parameters as world coordinates, making additive biases invariant to the absolute scale of the scene [S2]. For example, a residual $\Delta u_0$ is applied as $u_0' = u_0 + f \Delta u_0$ [S2].

## Operating characteristics
*   **Time Complexity:** The undistortion step uses Newton's method with a fixed number of iterations (typically 10) [S1, unofficial]. Each iteration involves constant-time arithmetic operations per pixel. Thus, the complexity is $O(N)$ where $N$ is the number of pixels, with a constant factor dependent on the number of iterations.
*   **Memory Complexity:** $O(N)$ for storing pixel coordinates and ray directions. The camera parameters are $O(1)$.
*   **Strengths:** Accurately models real-world lens distortions, improving reconstruction quality in neural rendering [S1, unofficial].
*   **Limitations:** The iterative undistortion is computationally more expensive than direct projection. Convergence is not guaranteed for extreme distortions, though 10 iterations are usually sufficient [S1, unofficial].
*   **When to use:** When high-fidelity camera calibration is required, especially for wide-angle or fisheye lenses [S1, unofficial].
*   **When not to use:** For simple pinhole cameras without distortion, or when performance is critical and distortion is negligible.

## Background
The camera model is implemented in the Nerfies library by Google [S1, unofficial]. It is based on standard computer vision camera models with radial and tangential distortion [S1, unofficial]. The Pixel Scale variant is described in the CamP paper for camera preconditioning in neural radiance fields [S2].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] CamP: Camera Preconditioning for Neural Radiance Fields, https://doi.org/10.1145/3618321, fetched 2026-10-30
