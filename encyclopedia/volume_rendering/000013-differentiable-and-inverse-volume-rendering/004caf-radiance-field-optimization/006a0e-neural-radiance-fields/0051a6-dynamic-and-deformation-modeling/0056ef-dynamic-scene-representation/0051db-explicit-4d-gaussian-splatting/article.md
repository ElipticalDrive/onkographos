# Explicit 4D Gaussian Splatting

Also called: Dynamic 3DGS with Keyframe Interpolation, Fully Explicit Dynamic Gaussian Splatting

## Overview
Explicit 4D Gaussian Splatting (Ex4DGS) is a method for dynamic novel view synthesis that achieves memory efficiency and rendering speed by separating scene primitives into static and dynamic categories and explicitly interpolating their attributes over time [S1]. Unlike implicit deformation fields or continuous trajectory models, Ex4DGS stores sparse keyframe data for dynamic Gaussians and uses specific interpolators to reconstruct continuous motion, handling complex phenomena like self-occlusion and object disappearance without requiring auxiliary masks or supervision [S1]. It builds on [Dynamic Scene Representation](15830-dynamic-scene-representation.md) by introducing a fully explicit temporal domain where attributes are sampled at discrete timestamps and interpolated [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of input images (video frames) and initial 3D points (e.g., from COLMAP on the first frame) [S1].
*   **Output:** A set of learnable Gaussian primitives defined over time $t \in [0, T]$, capable of rendering novel views at any timestamp within the interval [S1].

**Data Structures:**
The scene consists of $N$ Gaussians. Each Gaussian $i$ is classified as either Static ($G_s$) or Dynamic ($G_d$).

1.  **Static Gaussian ($G_s$):**
    *   **Fields:** Pivot position $x \in \mathbb{R}^3$, translation vector $d \in \mathbb{R}^3$, scale $S$, rotation $R$ (constant), density $\sigma$, color $c$ (view-dependent via spherical harmonics) [S1].
    *   **Layout:** $x$ and $d$ define the linear motion. $S, R, \sigma, c$ are constant over time [S1].

2.  **Dynamic Gaussian ($G_d$):**
    *   **Fields:** A sequence of keyframes $K = \{t | t = nI, n \in \mathbb{Z}, t \in T\}$, where $I$ is the time interval [S1].
    *   **Keyframe Data:** For each keyframe $n$, store position $p_n \in \mathbb{R}^3$, rotation $r_n$ (quaternion), and opacity parameters [S1].
    *   **Opacity Parameters:** Two Gaussians $g_{so}, g_{fo}$ with means $a_{so}, a_{of}$ and variances $b_{so}, b_{fo}$ [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize all $N$ Gaussian points as Static ($G_s$), assuming linear motion. The initial position is the 3D point from the first frame. The translation vector $d$ is initialized based on the initial estimate of motion [S1].
2.  **Static Motion Model:** For a static Gaussian, the position $\mu(t)$ at time $t$ is calculated using normalized time $t' = t/l$, where $l$ is the scene duration:
    $$ \mu(t) = x + t' d, \quad t' = \frac{t}{l} \in [0, 1] $$
    where $x$ is the pivot position and $d$ is the translation vector [S1].
3.  **Dynamic Classification (Motion-Based Trigger):**
    *   During training, measure the motion of each static Gaussian.
    *   Calculate the normalized translation magnitude: $\frac{\|d\|}{\|\lambda\|^2}$, where $\lambda$ is the distance between the point and the camera at the last timestamp [S1].
    *   Sort all Gaussians by this metric.
    *   Convert the top-$\eta$ percent of points (empirically $\eta = 2$) into Dynamic Gaussians ($G_d$) [S1].
    *   **Initialization of Converted $G_d$:**
        *   Estimate position $p_n$ at each keyframe using $x$ and $d$.
        *   Set rotation $r_n$ to the same value for all keyframes.
        *   Initialize opacity to be visible in all keyframes [S1].
4.  **Progressive Training:**
    *   Start training with a short initial duration (e.g., 10 frames) [S1].
    *   Increment the duration by the interval $I$ every specific step (e.g., every 400 iterations) [S1].
    *   When the number of keyframes increases, estimate the position and rotation of the Gaussian by linear regression using the last $\rho$ frames to share motion information with the next keyframe [S1].
5.  **Temporal Interpolation (Rendering):**
    *   **Position:** Use Cubic Hermite Interpolation (CHip) for position. Compute tangents $m_n, m_{n+1}$ from neighbor keyframe positions:
        $$ m_n = \frac{p_{n+1} - p_{n-1}}{2I}, \quad m_{n+1} = \frac{p_{n+2} - p_n}{2I} $$
        For $t$ in interval $[nI, (n+1)I]$, let $t' = \frac{t - nI}{I}$ and $n = \lfloor t/I \rfloor$. Then:
        $$ \mu(t) = CHip(p_n, m_n, p_{n+1}, m_{n+1}; t') $$
        where $CHip(p_0, m_0, p_1, m_1; t) = (2t^3 - 3t^2 + 1)p_0 + (t^3 - 2t^2 + t)m_0 + (-2t^3 + 3t^2)p_1 + (t^3 - t^2)m_1$ [S1].
    *   **Rotation:** Use Spherical Linear Interpolation (Slerp) for rotation quaternions $r_n, r_{n+1}$:
        $$ q(t) = Slerp(r_n, r_{n+1}; t') $$
        where $Slerp(x_0, x_1; t) = \frac{\sin[(t)\Omega]}{\sin \Omega}x_0 + \frac{\sin[(1 - t)\Omega]}{\sin \Omega}x_1$ and $\cos \Omega = x_0 \cdot x_1$ [S1].
    *   **Opacity:** Use a simplified Gaussian mixture model. Temporal opacity $\sigma_t(t)$ is:
        $$ \sigma_t(t) = \begin{cases} e^{-\frac{(t-a_{so})^2}{b_{so}}} & t < a_{so} \\ 1 & a_{so} \le t \le a_{of} \\ e^{-\frac{(t-a_{of})^2}{b_{fo}}} & t > a_{of} \end{cases} $$
        where $a_{so}, a_{of}$ are means and $b_{so}, b_{fo}$ are variances of the two Gaussians [S1].
6.  **Rendering:**
    *   Compute the 3D covariance $\Sigma$ from scale $S$ and rotation $R$: $\Sigma = R S S^\top R^\top$ [S1].
    *   Project to 2D: $\Sigma' = J W \Sigma W^\top J^\top$, where $J$ is the Jacobian of the affine approximation of perspective projection and $W$ is the viewing transformation [S1].
    *   Render color $C$ via alpha blending:
        $$ C = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i })c_i, \quad T_i = e^{-\sum_{j=1}^{i-1} \sigma_j \delta_j} $$
        where $\delta_i$ is the interval along the ray, and $i,j$ are depth-ordered visible Gaussians [S1].
7.  **Optimization and Pruning:**
    *   **Losses:** Use L1 loss and SSIM loss between rendered and ground truth images [S1].
    *   **Regularization:** Minimize $\|d\|$ for static points and $\|p_{n+1} - p_n\|$ for dynamic points (parameter 0.0001) [S1].
    *   **Point Backtracking:** Calculate error $E$ for pixel $k$:
        $$ E = \frac{\sum_{i} \sigma_i \times \prod_{j=1}^{i-1} (1 - \sigma_j) \times q_k}{\sum_{i} \sigma_i \times \prod_{j=1}^{i-1} (1 - \sigma_j)} $$
        where $q_k$ is the measured error (L1/SSIM) at pixel $k$, and $i,j$ are depth-ordered visible Gaussians [S1].
    *   Calculate accumulated error $E_{total} = \sum_{v \in D} \frac{E_v}{|D|}$ over training views $D$. Prune points with error $> E_{total}$ at pre-defined steps [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial.transform import Rotation

def cubic_hermite(p0, m0, p1, m1, t):
    """
    Cubic Hermite Interpolation for position.
    p0, p1: positions (np.array, shape (3,))
    m0, m1: tangents (np.array, shape (3,))
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
    q0, q1: quaternions (np.array, shape (4,))
    t: normalized time in [0, 1]
    """
    # Normalize quaternions
    q0 = q0 / np.linalg.norm(q0)
    q1 = q1 / np.linalg.norm(q1)
    
    dot = np.dot(q0, q1)
    
    # If dot product is negative, slerp won't take the shorter path,
    # so negate one quaternion.
    if dot < 0:
        q1 = -q1
        dot = -dot
        
    # Clamp
    dot = np.clip(dot, -1.0, 1.0)
    
    omega = np.arccos(dot)
    sin_omega = np.sin(omega)
    
    if sin_omega < 1e-6:
        return q0 # Linear interpolation if close
        
    ratio0 = np.sin((1 - t) * omega) / sin_omega
    ratio1 = np.sin(t * omega) / sin_omega
    
    return ratio0 * q0 + ratio1 * q1

def get_temporal_opacity(t, a_so, a_of, b_so, b_fo):
    """
    Simplified Gaussian mixture model for opacity.
    """
    if t < a_so:
        return np.exp(-((t - a_so)**2) / b_so)
    elif t > a_of:
        return np.exp(-((t - a_of)**2) / b_fo)
    else:
        return 1.0

def render_gaussian(gaussian, camera_params, time_t, interval_I, duration_l):
    """
    Renders a single Gaussian at time t.
    gaussian: dict with type ('static' or 'dynamic'), attributes
    camera_params: dict with projection matrix, view matrix, etc.
    time_t: current timestamp
    interval_I: keyframe interval
    duration_l: scene duration
    """
    if gaussian['type'] == 'static':
        # Static motion model
        t_prime = time_t / duration_l
        mu = gaussian['pivot'] + t_prime * gaussian['translation']
        R = gaussian['rotation'] # Constant
        S = gaussian['scale']
        sigma = gaussian['density']
        c = gaussian['color']
        
    elif gaussian['type'] == 'dynamic':
        # Dynamic interpolation
        n = int(time_t // interval_I)
        t_prime = (time_t - n * interval_I) / interval_I
        
        # Position via CHip
        p_n = gaussian['keyframes'][n]['pos']
        p_n1 = gaussian['keyframes'][n+1]['pos']
        # Tangents from neighbors
        p_n1_prev = gaussian['keyframes'][n-1]['pos'] if n > 0 else p_n
        p_n2 = gaussian['keyframes'][n+2]['pos'] if n+2 < len(gaussian['keyframes']) else p_n1
        
        m_n = (p_n1 - p_n1_prev) / (2 * interval_I)
        m_n1 = (p_n2 - p_n) / (2 * interval_I)
        
        mu = cubic_hermite(p_n, m_n, p_n1, m_n1, t_prime)
        
        # Rotation via Slerp
        r_n = gaussian['keyframes'][n]['rot']
        r_n1 = gaussian['keyframes'][n+1]['rot']
        q = slerp(r_n, r_n1, t_prime)
        R = Rotation.from_quat(q).as_matrix()
        
        S = gaussian['scale'] # Assuming scale is constant or interpolated similarly if needed
        
        # Opacity
        sigma = get_temporal_opacity(
            time_t, 
            gaussian['opacity_params']['a_so'], 
            gaussian['opacity_params']['a_of'], 
            gaussian['opacity_params']['b_so'], 
            gaussian['opacity_params']['b_fo']
        )
        
        c = gaussian['color']
        
    # Project to 2D
    # Sigma = R S S^T R^T
    S_mat = np.diag(S)
    Sigma = R @ S_mat @ S_mat.T @ R.T
    
    # Camera projection (simplified)
    # J is Jacobian of affine approximation, W is viewing transform
    # In practice, this involves camera extrinsics/intrinsics
    # Sigma_prime = J @ W @ Sigma @ W.T @ J.T
    # For implementation, use standard 3DGS projection routine
    Sigma_prime = project_to_2d(Sigma, camera_params)
    
    return {
        'mu_2d': project_mean(mu, camera_params),
        'Sigma_prime': Sigma_prime,
        'sigma': sigma,
        'color': c
    }

def train_ex4dgs(frames, initial_points):
    """
    Main training loop.
    """
    # 1. Initialize all as static
    gaussians = []
    for p in initial_points:
        gaussians.append({
            'type': 'static',
            'pivot': p.pos,
            'translation': np.zeros(3), # Initial estimate
            'rotation': np.eye(3),
            'scale': np.ones(3),
            'density': 1.0,
            'color': p.color
        })
        
    duration_l = 10 # Initial duration
    interval_I = 10
    iteration = 0
    
    while iteration < max_iterations:
        # 2. Progressive Training: Extend duration
        if iteration % 400 == 0 and iteration > 0:
            duration_l += interval_I
            # Re-estimate positions/rotations for new keyframes using linear regression
            # on last rho frames (implementation detail omitted for brevity)
            
        # 3. Classification: Convert top 2% dynamic
        if iteration % 1000 == 0: # Check periodically
            motions = []
            for g in gaussians:
                if g['type'] == 'static':
                    # Measure motion in image space
                    # lambda is distance to camera at last timestamp
                    lambda_dist = get_camera_distance(g['pivot'], last_camera_params)
                    motion_metric = np.linalg.norm(g['translation']) / (lambda_dist**2)
                    motions.append((motion_metric, g))
            
            motions.sort(key=lambda x: x[0], reverse=True)
            num_dynamic = int(len(motions) * 0.02)
            
            for i in range(num_dynamic):
                g = motions[i][1]
                # Convert to dynamic
                g['type'] = 'dynamic'
                # Initialize keyframes
                # Estimate pos at each keyframe using x and d
                # Set rotation same in all keyframes
                # Initialize opacity visible
                g['keyframes'] = create_keyframes(g['pivot'], g['translation'], duration_l, interval_I)
                g['opacity_params'] = {'a_so': 0, 'a_of': duration_l, 'b_so': 1, 'b_fo': 1}
                
        # 4. Render and Loss
        loss = 0
        for frame in frames:
            rendered = render_scene(gaussians, frame.camera_params, frame.time)
            gt = frame.image
            loss += l1_loss(rendered, gt) + ssim_loss(rendered, gt)
            
        # 5. Backprop and Update
        loss.backward()
        optimizer.step()
        
        # 6. Pruning
        if iteration % prune_step == 0:
            prune_gaussians(gaussians, threshold=E_total)
            
        iteration += 1
```

**Parameters:**
*   **Time Interval ($I$):** 10 frames [S1].
*   **Initial Duration:** 10 frames [S1].
*   **Duration Increment Step:** Every 400 iterations [S1].
*   **Dynamic Point Conversion Threshold ($\eta$):** 2 percent [S1].
*   **Regularization Parameters:** 0.0001 for both static ($\|d\|$) and dynamic ($\|p_{n+1} - p_n\|$) [S1].
*   **Optimizer:** RAdam [S1].

**Pitfalls:**
*   **Disappearing/Reappearing Objects:** Self-occlusion can cause objects to disappear and reappear. Previous methods require multiple Gaussians for this, increasing memory. Ex4DGS handles this via the simplified Gaussian mixture model for opacity and point-backtracking to prune redundant points [S1].
*   **Rotation Bias:** Linear interpolation of rotation angles causes bias. Slerp is required [S1].
*   **Overfitting/Over-smoothing:** Linear interpolation for position may not capture complex movements. CHip is used to avoid this without additional storage [S1].

## Operating characteristics
*   **Time Complexity:** Rendering speed is comparable to static 3DGS because interpolation is computed per Gaussian during rasterization, but the number of Gaussians is reduced by separating static/dynamic and pruning. Measured at 62 fps on a single NVIDIA 2080Ti GPU for 300-frame scenes with 1352×1014 resolution [S1].
*   **Memory Complexity:** Memory efficient. Static Gaussians store only linear motion parameters ($x, d$). Dynamic Gaussians store sparse keyframes. No auxiliary encoding modules are needed [S1].
*   **Strengths:** Handles sparse initial point clouds well via progressive training. No need for auxiliary masks or supervision for static/dynamic separation. Handles complex motions and occlusions effectively.
*   **Limitations:** Requires uniform keyframe intervals for simplicity. Performance depends on the accuracy of the initial point cloud (COLMAP) [S1].

## Background
Explicit 4D Gaussian Splatting (Ex4DGS) was introduced to address the limitations of existing dynamic novel view synthesis methods, which often suffer from slow rendering speeds due to reliance on neural networks or high memory costs due to continuous trajectory models [S1]. The method builds on the static 3D Gaussian Splatting framework [13] and extends it with explicit temporal interpolation. It was validated on the Neural 3D Video dataset and the Technicolor dataset [S1].

## Sources
[S1] Fully Explicit Dynamic Gaussian Splatting, https://doi.org/10.52202/079017-0174, fetched 2026-10-26
