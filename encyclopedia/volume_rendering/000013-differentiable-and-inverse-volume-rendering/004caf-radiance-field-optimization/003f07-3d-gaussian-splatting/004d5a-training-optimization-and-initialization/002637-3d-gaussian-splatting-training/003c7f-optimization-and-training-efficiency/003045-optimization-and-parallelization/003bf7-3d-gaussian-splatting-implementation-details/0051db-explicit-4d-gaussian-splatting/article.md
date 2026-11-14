# Explicit 4D Gaussian Splatting

Also called: Dynamic 3DGS with Keyframe Interpolation, Fully Explicit Dynamic Gaussian Splatting

## Overview
Explicit 4D Gaussian Splatting (Ex4DGS) is a method for dynamic scene rendering that achieves memory efficiency and rendering capacity by separating Gaussians into static and dynamic categories and explicitly sampling positions and rotations at sparse timestamps [S1]. It differs from implicit trajectory-based methods by storing explicit keyframe data for dynamic Gaussians and using interpolation to reconstruct continuous motion, thereby avoiding the memory overhead of multiple primitives for occluded objects [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Sparse 3D point cloud from the first frame (e.g., from COLMAP) [S1].
*   A sequence of input images (video frames) with known camera poses [S1].
*   Hyperparameters: time interval $I$, initial duration $L_{init}$, duration increment step, dynamic conversion threshold $\eta$, regularization weight $\lambda_{reg}$, and regression window $\rho$ [S1].

**Outputs:**
*   A set of static Gaussians $G_s$ with parameters $\{x, d, S, R, c, \sigma\}$ [S1].
*   A set of dynamic Gaussians $G_d$ with keyframe parameters $\{p_n, r_n, \sigma_n, S_n, c_n\}$ for each keyframe $n$ [S1].
*   Rendered images for any timestamp $t \in [0, L]$ [S1].

### Data Structures

**Static Gaussian ($G_s$):**
*   `pivot_position` ($x$): 3D vector, initial position.
*   `translation_vector` ($d$): 3D vector, linear motion direction.
*   `scale` ($S$): 3D vector (or 9D for full covariance, but typically decomposed).
*   `rotation` ($R$): Quaternion or 3x3 matrix.
*   `color` ($c$): Spherical Harmonics coefficients.
*   `opacity` ($\sigma$): Scalar.

**Dynamic Gaussian ($G_d$):**
*   `keyframes`: List of states at timestamps $t = nI$.
*   `keyframe_position` ($p_n$): 3D vector at keyframe $n$.
*   `keyframe_rotation` ($r_n$): Quaternion at keyframe $n$.
*   `keyframe_opacity` ($\sigma_n$): Scalar at keyframe $n$.
*   `keyframe_scale` ($S_n$): 3D vector at keyframe $n$.
*   `keyframe_color` ($c_n$): SH coefficients at keyframe $n$.
*   *Note:* Tangents $m_n$ are not stored but computed on-the-fly from neighbor positions [S1].

### Numbered Steps

1.  **Initialization:** Initialize all Gaussian points from the input point cloud as static. Assume linear motion for all points initially [S1].
2.  **Static Motion Model:** For static Gaussians, position $\mu(t)$ is defined by a pivot $x$ and translation $d$:
    $$ \mu(t) = x + t' d, \quad t' = \frac{t}{l} \in [0, 1] $$
    where $l$ is the scene duration [S1].
3.  **Progressive Training:**
    *   Start training with a short initial duration $L_{init}$ (e.g., 10 frames) [S1].
    *   Increment the duration by the interval $I$ every specific number of iterations (e.g., 400 iterations) [S1].
    *   When the duration increases, estimate new keyframe positions/rotations using linear regression over the last $\rho$ frames to share motion information [S1].
4.  **Static/Dynamic Separation:**
    *   Measure motion in image space for static points. Calculate normalized translation:
        $$ \text{motion} = \frac{\|d\|}{\|\lambda\|^2} $$
        where $\lambda$ is the distance between the point and the camera at the last timestamp [S1].
    *   Sort points by this metric. Convert the top-$\eta$ percent (empirically $\eta=2$) into dynamic Gaussians [S1].
    *   For converted dynamic points, estimate position at each keyframe using $x$ and $d$. Initialize rotation to the same value in all keyframes and opacity to visible [S1].
5.  **Dynamic Interpolation:** For a dynamic Gaussian at time $t$:
    *   Determine keyframe index $n = \lfloor t/I \rfloor$ and normalized time $t' = (t - nI)/I$ [S1].
    *   **Position:** Use Cubic Hermite Interpolation (CHip). Tangents are derived from neighbor positions:
        $$ m_n = \frac{p_{n+1} - p_{n-1}}{2I}, \quad m_{n+1} = \frac{p_{n+2} - p_n}{2I} $$
        $$ \mu(t) = CHip(p_n, m_n, p_{n+1}, m_{n+1}; t') $$
        where $CHip(p_0, m_0, p_1, m_1; t) = (2t^3 - 3t^2 + 1)p_0 + (t^3 - 2t^2 + t)m_0 + (-2t^3 + 3t^2)p_1 + (t^3 - t^2)m_1$ [S1].
    *   **Rotation:** Use Spherical Linear Interpolation (Slerp) on quaternions:
        $$ q(t) = Slerp(r_n, r_{n+1}; t') $$
        where $Slerp(x_0, x_1; t) = \frac{\sin[(t)\Omega]}{\sin \Omega}x_0 + \frac{\sin[(1 - t)\Omega]}{\sin \Omega}x_1$ and $\cos \Omega = x_0 \cdot x_1$ [S1].
    *   **Opacity:** Use a simplified Gaussian mixture model to handle appearance/disappearance:
        $$ \sigma_t(t) = \begin{cases} e^{-\frac{(t-a_{so})^2}{b_{so}}} & t < a_{so} \\ 1 & a_{so} \le t \le a_{of} \\ e^{-\frac{(t-a_{of})^2}{b_{fo}}} & t > a_{of} \end{cases} $$
        where $a_{so}, a_{of}$ are means and $b_{so}, b_{fo}$ are variances of two Gaussians [S1].
6.  **Rendering:** Render using standard 3DGS differentiable rasterization. Color $C$ is computed via alpha blending:
    $$ C = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i })c_i, \quad T_i = e^{-\sum_{j=1}^{i-1} \sigma_j \delta_j} $$
    where $\sigma_i$ is the density (derived from opacity and scale), $\delta_i$ is the interval, and $c_i$ is color [S1].
7.  **Optimization and Pruning:**
    *   **Losses:** Use L1 loss and SSIM loss between rendered and ground truth images [S1].
    *   **Regularization:** Minimize $\|d\|$ for static points and $\|p_{n+1} - p_n\|$ for dynamic points with weight 0.0001 [S1].
    *   **Point Backtracking:** Calculate per-pixel error $E$ for pixel $k$:
        $$ E = \frac{\sum_{i} \sigma_i \times \prod_{j=1}^{i-1} (1 - \sigma_j) \times q_k}{\sum_{i} \sigma_i \times \prod_{j=1}^{i-1} (1 - \sigma_j)} $$
        where $q_k$ is the measured error (L1/SSIM) at pixel $k$ [S1].
    *   Calculate accumulated error $E_{total} = \sum_{v \in D} \frac{E_v}{|D|}$ over training views $D$ [S1].
    *   Prune Gaussians with error exceeding $E_{total}$ at pre-defined steps [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial.transform import Rotation

def cubic_hermite_interpolation(p0, m0, p1, m1, t):
    """
    Cubic Hermite Interpolation for position.
    p0, p1: positions (3,)
    m0, m1: tangents (3,)
    t: normalized time in [0, 1]
    """
    t2 = t * t
    t3 = t2 * t
    h00 = 2 * t3 - 3 * t2 + 1
    h10 = t3 - 2 * t2 + t
    h01 = -2 * t3 + 3 * t2
    h11 = t3 - t2
    return h00 * p0 + h10 * m0 + h01 * p1 + h11 * m1

def slerp(q0, q1, t):
    """
    Spherical Linear Interpolation for quaternions.
    q0, q1: quaternions (4,)
    t: normalized time in [0, 1]
    """
    # Normalize quaternions
    q0 = q0 / np.linalg.norm(q0)
    q1 = q1 / np.linalg.norm(q1)
    
    dot = np.dot(q0, q1)
    
    # If negative dot, invert one quaternion
    if dot < 0:
        q1 = -q1
        dot = -dot
        
    # Clamp
    dot = np.clip(dot, -1.0, 1.0)
    
    theta = np.arccos(dot)
    sin_theta = np.sin(theta)
    
    if sin_theta < 1e-6:
        return q0 # Linear interpolation if close
        
    s0 = np.sin((1 - t) * theta) / sin_theta
    s1 = np.sin(t * theta) / sin_theta
    
    return s0 * q0 + s1 * q1

def get_dynamic_position(pn, pn_minus_1, pn_plus_1, pn_plus_2, I, t):
    """
    Get position for dynamic Gaussian at time t.
    pn: position at keyframe n
    I: interval
    t: current time
    """
    n = int(t // I)
    t_prime = (t - n * I) / I
    
    # Calculate tangents
    mn = (pn_plus_1 - pn_minus_1) / (2 * I)
    mn_plus_1 = (pn_plus_2 - pn) / (2 * I)
    
    return cubic_hermite_interpolation(pn, mn, pn_plus_1, mn_plus_1, t_prime)

def get_dynamic_rotation(rn, rn_plus_1, I, t):
    """
    Get rotation for dynamic Gaussian at time t.
    rn: quaternion at keyframe n
    I: interval
    t: current time
    """
    n = int(t // I)
    t_prime = (t - n * I) / I
    return slerp(rn, rn_plus_1, t_prime)

def get_dynamic_opacity(t, a_so, b_so, a_of, b_of):
    """
    Simplified Gaussian mixture model for opacity.
    """
    if t < a_so:
        return np.exp(-((t - a_so)**2) / b_so)
    elif t <= a_of:
        return 1.0
    else:
        return np.exp(-((t - a_of)**2) / b_of)

def render_gaussians(gaussians, camera_pose, image_shape):
    """
    Standard 3DGS rasterization.
    """
    # Project Gaussians to 2D
    # Compute covariance in camera space: Sigma' = J W Sigma W^T J^T
    # Rasterize using alpha blending
    pass

def train_ex4dgs(points, images, camera_poses, hyperparams):
    """
    Main training loop.
    """
    # 1. Initialize all as static
    static_gaussians = initialize_static(points)
    dynamic_gaussians = []
    
    duration = hyperparams['initial_duration']
    interval = hyperparams['interval']
    
    for iteration in range(hyperparams['max_iterations']):
        # 2. Progressive Training: Extend duration
        if iteration % hyperparams['duration_step'] == 0:
            duration += interval
            # Estimate new keyframes using linear regression on last rho frames
            update_keyframes(static_gaussians, dynamic_gaussians, duration, hyperparams['rho'])
            
        # 3. Separate Static/Dynamic
        if iteration % hyperparams['separation_step'] == 0:
            motion_scores = []
            for g in static_gaussians:
                # Normalize translation by distance to camera at last timestamp
                dist = np.linalg.norm(g.pivot_position - camera_poses[-1].position)
                score = np.linalg.norm(g.translation_vector) / (dist**2)
                motion_scores.append((score, g))
            
            motion_scores.sort(key=lambda x: x[0], reverse=True)
            num_dynamic = int(len(motion_scores) * hyperparams['eta'] / 100)
            
            for i in range(num_dynamic):
                g = motion_scores[i][1]
                # Convert to dynamic
                # Estimate position at each keyframe using x and d
                # Initialize rotation and opacity
                dynamic_gaussians.append(convert_to_dynamic(g, duration, interval))
                static_gaussians.remove(g)
                
        # 4. Optimization Step
        # Sample a random timestamp t in [0, duration]
        t = np.random.uniform(0, duration)
        
        # Get Gaussian states at t
        current_static = []
        for g in static_gaussians:
            mu = g.pivot_position + (t / duration) * g.translation_vector
            current_static.append({**g, 'position': mu})
            
        current_dynamic = []
        for g in dynamic_gaussians:
            pos = get_dynamic_position(g.keyframe_positions, ...) # Access neighbors
            rot = get_dynamic_rotation(g.keyframe_rotations, ...)
            op = get_dynamic_opacity(t, ...)
            current_dynamic.append({**g, 'position': pos, 'rotation': rot, 'opacity': op})
            
        all_gaussians = current_static + current_dynamic
        
        # Render
        rendered_image = render_gaussians(all_gaussians, camera_poses[current_frame], image_shape)
        gt_image = images[current_frame]
        
        # Loss
        loss = l1_loss(rendered_image, gt_image) + ssim_loss(rendered_image, gt_image)
        
        # Regularization
        reg_loss = 0
        for g in static_gaussians:
            reg_loss += np.linalg.norm(g.translation_vector)
        for g in dynamic_gaussians:
            # Sum of distances between adjacent keyframes
            for i in range(len(g.keyframe_positions) - 1):
                reg_loss += np.linalg.norm(g.keyframe_positions[i+1] - g.keyframe_positions[i])
                
        total_loss = loss + hyperparams['reg_weight'] * reg_loss
        
        # Backprop
        optimizer.step(total_loss)
        
        # 5. Pruning
        if iteration % hyperparams['prune_step'] == 0:
            # Calculate accumulated error E_total
            # Prune high error Gaussians
            prune_gaussians(all_gaussians, hyperparams['prune_threshold'])
```

### Parameters
*   **Time Interval ($I$):** 10 frames [S1].
*   **Initial Duration:** 10 frames [S1].
*   **Duration Increment Step:** 400 iterations [S1].
*   **Regularization Weight:** 0.0001 for both static and dynamic [S1].
*   **Dynamic Conversion Threshold ($\eta$):** 2 percent [S1].
*   **Optimizer:** RAdam [S1].

### Edge Cases and Pitfalls
*   **Disappearing/Reappearing Objects:** Self-occlusion can cause objects to disappear and reappear. Previous methods require multiple Gaussians for this, increasing memory. Ex4DGS handles this via the simplified Gaussian mixture model for opacity and point-backtracking to prune redundant points [S1].
*   **Sparse Point Clouds:** Initial sparse points may lead to local minima. The progressive training scheme (starting with short duration) prevents this [S1].
*   **Rotation Bias:** Linear interpolation of rotations causes bias. Slerp is used to avoid this [S1].

## Subtopics

### Continuous Attenuation Model for 3D Gaussian Splatting
Focuses on density attenuation during rendering. [Link](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)

### DAVIGS
Another dynamic Gaussian Splatting method, likely using different interpolation or representation. [Link](12021-davigs.md)

### Sorting and Rendering Pipeline Optimization
Optimizes the rasterization order and pipeline. [Link](12180-sorting-and-rendering-pipeline-optimization.md)

### DeSplat
Method for mesh extraction or simplification. [Link](12925-desplat.md)

### 360 Gaussian Splatting
Handles omnidirectional scenes. [Link](12929-360-gaussian-splatting.md)

### Voxel-based Gaussian Splatting
Uses voxel structures for acceleration. [Link](13071-voxel-based-gaussian-splatting.md)

### SOGS
Second-order anchor optimization. [Link](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)

### 3iGS
Factorised tensorial illumination. [Link](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)

### GVKF
Gaussian Voxel Kernel Functions. [Link](13979-gaussian-voxel-kernel-functions-gvkf.md)

### SWAG
Appearance-conditioned Gaussians for wild images. [Link](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md)

### TraGraph-GS
Graph-based trajectory modeling. [Link](14454-tragraph-gs.md)

## Operating characteristics

*   **Time Complexity:** Rendering speed is comparable to static 3DGS, as interpolation is performed per Gaussian per frame. Training time includes the overhead of keyframe management and progressive learning.
*   **Memory Complexity:** Reduced compared to methods storing full trajectory functions or multiple primitives for occlusions. Only keyframe data is stored for dynamic Gaussians [S1].
*   **Measured Performance:** 62 fps on a single NVIDIA 2080Ti GPU for 300-frame scenes with 1352×1014 resolution [S1].
*   **Strengths:** Memory efficient, handles dynamic scenes with sparse initial points, avoids bias in rotation interpolation, handles disappearing/reappearing objects well.
*   **Limitations:** Requires uniform keyframe intervals (assumed for simplicity), performance depends on the quality of initial point clouds.
*   **When to Use:** Dynamic novel view synthesis where memory and rendering speed are critical, and initial sparse point clouds are available.
*   **When Not to Use:** Scenes with extremely complex, non-smooth motions that cannot be well-approximated by cubic Hermite interpolation, or where dense initial tracking is not feasible.

## Background

*   **Origin:** Ex4DGS was introduced in the paper "Fully Explicit Dynamic Gaussian Splatting" [S1].
*   **History:** It builds on the 3D Gaussian Splatting (3DGS) framework [S1]. It addresses limitations in previous dynamic 3DGS methods that relied on implicit trajectory functions or required multiple Gaussians for occluded objects, leading to higher memory and computational costs [S1].
*   **Usage:** Validated on Neural 3D Video dataset and Technicolor dataset [S1].

## Sources

[S1] Fully Explicit Dynamic Gaussian Splatting, https://doi.org/10.52202/079017-0174, fetched 2026-10-26
