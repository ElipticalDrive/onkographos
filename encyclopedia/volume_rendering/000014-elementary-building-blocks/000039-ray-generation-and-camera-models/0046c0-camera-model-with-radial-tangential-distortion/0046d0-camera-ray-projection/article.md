# Camera Ray Projection
Also called: Inverse Projection with Distortion

## Overview
Camera Ray Projection solves the inverse camera distortion problem: it maps distorted normalized pixel coordinates to undistorted normalized coordinates, which are then used to construct 3D world-space rays. It differs from forward projection by solving a non-linear system of equations via Newton-Raphson iteration rather than applying a direct algebraic transformation [S1, S2, unofficial]. This method is essential for accurate ray generation in volume rendering and NeRF pipelines when using cameras with significant radial and tangential lens distortion [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Distorted normalized coordinates $(x_d, y_d)$ [S2, unofficial].
*   Camera intrinsic parameters: focal lengths ($f_x, f_y$), principal point $(u_0, v_0)$, skew $s$, and pixel aspect ratio [S2, unofficial].
*   Distortion coefficients: radial ($k_1, k_2, k_3$) and tangential ($p_1, p_2$) [S2, unofficial].
*   Camera extrinsic parameters: orientation matrix $R$ (3x3) and position $C$ (3x1) [S2, unofficial].
*   Iteration parameters: `max_iterations` (default 10) [S2, unofficial], `eps` (default $10^{-9}$) [S2, unofficial].

**Outputs:**
*   Undistorted normalized coordinates $(x, y)$ [S2, unofficial].
*   World-space ray direction vector $\mathbf{d}_{world}$ [S2, unofficial].

### Data Structures
The method operates on scalar or vectorized arrays of coordinates. No complex data structures are required beyond standard floating-point arrays for coordinates and coefficients.

### Algorithm Steps
1.  **Coordinate Normalization**: Convert pixel coordinates $(u, v)$ to distorted normalized coordinates $(x_d, y_d)$ using intrinsic parameters [S2, unofficial]:
    $$
    y_d = \frac{v - v_0}{f_y}
    $$
    $$
    x_d = \frac{u - u_0 - s \cdot y_d}{f_x}
    $$
    where $v_0, u_0$ are principal points, $f_x, f_y$ are focal lengths in pixel units, and $s$ is skew [S2, unofficial].

2.  **Initialization**: Initialize undistorted coordinates $(x, y)$ with the distorted values:
    $$
    x = x_d, \quad y = y_d
    $$
    [S1, S2, unofficial].

3.  **Newton-Raphson Iteration**: Iterate up to `max_iterations` times [S2, unofficial]:
    a.  **Compute Radial Distance**:
        $$
        r^2 = x^2 + y^2
        $$
        [S1, S2, unofficial].

    b.  **Compute Distortion Factor**:
        $$
        d = 1 + k_1 r^2 + k_2 r^4 + k_3 r^6
        $$
        [S2, unofficial]. Note: Some implementations use a rational function for radial distortion, but the standard implementation cited uses the polynomial form [S1, S2, unofficial].

    c.  **Compute Residuals**: The goal is to find $(x, y)$ such that the forward distortion model yields $(x_d, y_d)$. The residuals are defined as the difference between the computed distorted coordinates and the input distorted coordinates [S1, S2, unofficial]:
        $$
        f_x = d \cdot x + 2 p_1 x y + p_2 (r^2 + 2 x^2) - x_d
        $$
        $$
        f_y = d \cdot y + 2 p_2 x y + p_1 (r^2 + 2 y^2) - y_d
        $$
        [S1, S2, unofficial].

    d.  **Compute Jacobian**: Calculate the partial derivatives of the residuals with respect to $x$ and $y$ [S1, S2, unofficial].
        Let $d_r = \frac{d(r^2)}{d(r^2)} = k_1 + 2 k_2 r^2 + 3 k_3 r^4$ [S1, unofficial].
        The Jacobian elements are:
        $$
        J_{xx} = \frac{\partial f_x}{\partial x} = d + x \frac{\partial d}{\partial x} + 2 p_1 y + p_2 (2x + 4x) \quad \text{(simplified below)}
        $$
        More precisely, using the chain rule and $r^2 = x^2 + y^2$:
        $$
        J_{xx} = d + 2 x^2 d_r + 2 p_1 y + 6 p_2 x
        $$
        $$
        J_{xy} = \frac{\partial f_x}{\partial y} = 2 x y d_r + 2 p_1 x + 2 p_2 y
        $$
        $$
        J_{yx} = J_{xy}
        $$
        $$
        J_{yy} = d + 2 y^2 d_r + 6 p_1 y + 2 p_2 x
        $$
        [S1, unofficial].

    e.  **Solve for Step**: Use Cramer's rule to solve the linear system $J \cdot \Delta = -F$ for the step $(\Delta x, \Delta y)$ [S1, S2, unofficial].
        $$
        \text{det} = J_{xx} J_{yy} - J_{xy} J_{yx}
        $$
        [S1, S2, unofficial].
        If $|\text{det}| < \text{eps}$, the Jacobian is singular. Clamp the step to zero to prevent numerical blowup [S2, unofficial].
        Otherwise:
        $$
        \Delta x = \frac{f_x J_{yy} - f_y J_{xy}}{\text{det}}
        $$
        $$
        \Delta y = \frac{f_y J_{xx} - f_x J_{yx}}{\text{det}}
        $$
        [S1, S2, unofficial]. Note: Some implementations define the step as the solution to $J \Delta = -F$, resulting in negative signs in the numerator compared to solving $J \Delta = F$. The update rule $x_{new} = x_{old} + \Delta x$ implies $\Delta x$ is the correction term [S2, unofficial].

    f.  **Trust Region (Optional)**: Some implementations apply a trust region constraint to the step size to ensure stability [S1, unofficial].
        $$
        \text{stepSqr} = \Delta x^2 + \Delta y^2
        $$
        $$
        \text{radiusSqr} = \max(r^2 \cdot k_{RelStepRadius}^2, k_{StepRadius}^2)
        $$
        If $\text{stepSqr} > \text{radiusSqr}$, scale the step:
        $$
        \Delta x \leftarrow \Delta x \cdot \sqrt{\frac{\text{radiusSqr}}{\text{stepSqr}}}, \quad \Delta y \leftarrow \Delta y \cdot \sqrt{\frac{\text{radiusSqr}}{\text{stepSqr}}}
        $$
        [S1, unofficial].

    g.  **Update**:
        $$
        x \leftarrow x + \Delta x
        $$
        $$
        y \leftarrow y + \Delta y
        $$
        [S1, S2, unofficial].

    h.  **Convergence Check**: If $\text{stepSqr} < k_{MinStepSquaredNorm}$ (default $10^{-10}$), break and mark as converged [S1, unofficial].

4.  **Ray Direction Construction**: Convert undistorted normalized coordinates $(x, y)$ to local camera-space ray direction [S2, unofficial]:
    $$
    \mathbf{d}_{local} = \begin{bmatrix} x \\ y \\ 1 \end{bmatrix}
    $$
    Normalize $\mathbf{d}_{local}$:
    $$
    \mathbf{d}_{local} \leftarrow \frac{\mathbf{d}_{local}}{\|\mathbf{d}_{local}\|}
    $$
    [S2, unofficial].

5.  **World Space Transformation**: Transform the local ray direction to world coordinates using the camera orientation matrix $R$ [S2, unofficial]:
    $$
    \mathbf{d}_{world} = R \cdot \mathbf{d}_{local}
    $$
    Normalize $\mathbf{d}_{world}$:
    $$
    \mathbf{d}_{world} \leftarrow \frac{\mathbf{d}_{world}}{\|\mathbf{d}_{world}\|}
    $$
    [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def camera_ray_projection(pixels, camera_params):
    """
    pixels: array of shape (N, 2) with pixel coordinates (u, v)
    camera_params: object containing intrinsic, extrinsic, and distortion parameters
    """
    # 1. Normalize to distorted coordinates
    u = pixels[:, 0]
    v = pixels[:, 1]
    
    # Intrinsic parameters
    fx = camera_params.focal_length_x
    fy = camera_params.focal_length_y
    u0 = camera_params.principal_point_x
    v0 = camera_params.principal_point_y
    skew = camera_params.skew
    
    # Distorted normalized coordinates
    xd = (u - u0 - skew * (v - v0) / fy) / fx
    yd = (v - v0) / fy
    
    # 2. Initialize undistorted coordinates
    x = xd.copy()
    y = yd.copy()
    
    # Distortion coefficients
    k1, k2, k3 = camera_params.radial_distortion
    p1, p2 = camera_params.tangential_distortion
    
    max_iterations = camera_params.max_iterations
    eps = camera_params.eps
    
    # 3. Newton-Raphson Iteration
    for _ in range(max_iterations):
        # Compute r^2
        r2 = x**2 + y**2
        
        # Compute distortion factor d = 1 + k1*r^2 + k2*r^4 + k3*r^6
        d = 1.0 + r2 * (k1 + r2 * (k2 + r2 * k3))
        
        # Compute derivatives of d with respect to r^2
        # d(r^2) = k1 + 2*k2*r^2 + 3*k3*r^4
        dr2 = k1 + r2 * (2.0 * k2 + 3.0 * k3 * r2)
        
        # Compute residuals
        # fx = d*x + 2*p1*x*y + p2*(r^2 + 2*x^2) - xd
        # fy = d*y + 2*p2*x*y + p1*(r^2 + 2*y^2) - yd
        fx = d * x + 2.0 * p1 * x * y + p2 * (r2 + 2.0 * x**2) - xd
        fy = d * y + 2.0 * p2 * x * y + p1 * (r2 + 2.0 * y**2) - yd
        
        # Compute Jacobian elements
        # Jxx = d + 2*x^2*dr2 + 2*p1*y + 6*p2*x
        # Jxy = 2*x*y*dr2 + 2*p1*x + 2*p2*y
        # Jyx = Jxy
        # Jyy = d + 2*y^2*dr2 + 6*p1*y + 2*p2*x
        Jxx = d + 2.0 * x**2 * dr2 + 2.0 * p1 * y + 6.0 * p2 * x
        Jxy = 2.0 * x * y * dr2 + 2.0 * p1 * x + 2.0 * p2 * y
        Jyx = Jxy
        Jyy = d + 2.0 * y**2 * dr2 + 6.0 * p1 * y + 2.0 * p2 * x
        
        # Compute determinant
        det = Jxx * Jyy - Jxy * Jyx
        
        # Solve for step using Cramer's rule
        # Step_x = (fx * Jyy - fy * Jxy) / det
        # Step_y = (fy * Jxx - fx * Jyx) / det
        # Handle singular Jacobian
        step_x = np.where(np.abs(det) > eps, (fx * Jyy - fy * Jxy) / det, 0.0)
        step_y = np.where(np.abs(det) > eps, (fy * Jxx - fx * Jyx) / det, 0.0)
        
        # Update coordinates
        x = x + step_x
        y = y + step_y
        
        # Check convergence (optional, based on step size)
        step_sqr = step_x**2 + step_y**2
        if np.all(step_sqr < 1e-10):
            break
            
    # 4. Construct local ray direction
    # Stack x, y, 1
    local_dirs = np.stack([x, y, np.ones_like(x)], axis=-1)
    
    # Normalize
    norms = np.linalg.norm(local_dirs, axis=-1, keepdims=True)
    local_dirs = local_dirs / norms
    
    # 5. Transform to world space
    # R is 3x3 orientation matrix
    world_dirs = np.matmul(camera_params.orientation, local_dirs.T).T
    
    # Normalize world directions
    world_norms = np.linalg.norm(world_dirs, axis=-1, keepdims=True)
    world_dirs = world_dirs / world_norms
    
    return world_dirs
```

### Parameters
*   `max_iterations`: Maximum number of Newton-Raphson iterations. Default: 10 [S2, unofficial].
*   `eps`: Epsilon threshold for Jacobian determinant check to avoid division by zero. Default: $10^{-9}$ [S2, unofficial].
*   `k1, k2, k3`: Radial distortion coefficients. Default: 0.0 [S2, unofficial].
*   `p1, p2`: Tangential distortion coefficients. Default: 0.0 [S2, unofficial].
*   `kNumIterations`: Alternative max iterations parameter. Default: 100 [S1, unofficial].
*   `kMinStepSquaredNorm`: Convergence threshold for step size. Default: $10^{-10}$ [S1, unofficial].
*   `kRelStepRadius`, `kStepRadius`: Trust region parameters. Default: 0.1 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Singular Jacobian**: If the Jacobian determinant is near zero ($|\text{det}| < \text{eps}$), the Newton step is clamped to zero to prevent numerical blowup [S2, unofficial]. This effectively halts convergence for that iteration [S2, unofficial].
*   **Division by Zero**: Checks for $|\text{den}| < 10^{-12}$ and $|\text{det}| < 10^{-12}$ are used to break iteration early [S1, unofficial].
*   **Non-Finite Results**: Final check ensures result is finite: `converged && std::isfinite(x) && std::isfinite(y)` [S1, unofficial].
*   **Precision**: The method relies on floating-point arithmetic. Near-singular Jacobians can lead to large steps if not clamped [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot I)$, where $N$ is the number of pixels and $I$ is the number of Newton iterations. Each iteration involves constant-time arithmetic operations [S1, S2, unofficial].
*   **Memory Complexity**: $O(N)$ for storing coordinates and residuals [S2, unofficial].
*   **Performance**: The method is computationally intensive due to the iterative nature of Newton-Raphson. However, convergence is typically fast (10-20 iterations) for moderate distortion [S2, unofficial].
*   **Strengths**: Accurate inversion of complex distortion models; handles both radial and tangential distortion [S2, unofficial].
*   **Limitations**: Iterative solver may not converge for extreme distortion or poor initial guesses; computational cost is higher than direct algebraic methods [S1, S2, unofficial].
*   **When to Use**: When high accuracy is required for cameras with significant lens distortion [S2, unofficial].
*   **When Not to Use**: For real-time applications with minimal distortion, where a direct approximation or lookup table might be faster [S1, unofficial].

## Background
The method originates from the need to accurately model camera distortion in computer vision and graphics applications. The Newton-Raphson approach for inverting distortion models is a standard technique in photogrammetry and computer vision [S1, S2, unofficial]. The specific implementation details, including trust region constraints and Jacobian handling, are derived from modern implementations in NeRF and 3D reconstruction pipelines [S1, S2, unofficial].

## Sources
[S1] OpenSplat – 3D gaussian splatting in C++ with CPU and GPU support, https://github.com/pierotofy/OpenSplat, fetched 2026-10-26 (unofficial)
[S2] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
