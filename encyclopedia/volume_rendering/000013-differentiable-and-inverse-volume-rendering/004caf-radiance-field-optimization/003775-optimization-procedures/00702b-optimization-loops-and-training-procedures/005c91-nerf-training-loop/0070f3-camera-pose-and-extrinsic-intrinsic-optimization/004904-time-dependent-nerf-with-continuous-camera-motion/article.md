# Time-dependent NeRF with Continuous Camera Motion

Also called: COPE-NeRF training, Continuous Optimization of NeRF and poses, Joint Optimization Pipeline, Joint Optimization of Neural Radiance Fields and Continuous Camera Motion

## Overview
This method jointly optimizes a time-dependent Neural Radiance Field (NeRF) and continuous camera motion from monocular video, representing the scene as local Signed Distance Field (SDF) and appearance fields at each time step [S1]. Unlike methods that train a single global NeRF mapped via noisy initial transformations, this approach uses local time-dependent representations constrained by consistency losses to avoid local minima [S1]. It builds on [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md) by modeling camera motion as continuous velocities rather than discrete pose parameters.

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of monocular RGB images (video frames) with normalized timesteps $t \in [-1, 1]$. No initial camera poses or intrinsics are required [S1].
*   **Output:** A time-dependent NeRF model ($\phi_g, \phi_c$) representing scene geometry and appearance, and a continuous camera motion network ($\phi_v$) providing angular and linear velocities. Derived camera poses for each frame and the full scene geometry in a world coordinate system [S1].

**Data Structures:**
1.  **Time-dependent SDF Network ($\phi_g$):** An MLP taking 3D point $x \in \mathbb{R}^3$ and time $t$ as input, outputting scalar SDF value $s \in \mathbb{R}$ [S1].
2.  **Time-dependent Appearance Network ($\phi_c$):** An MLP taking 3D point $x \in \mathbb{R}^3$, viewing direction $d \in \mathbb{R}^2$ (in camera coordinates), and time $t$ as input, outputting color $c \in \mathbb{R}^3$ [S1].
3.  **Camera Motion Network ($\phi_v$):** An MLP taking time $t$ as input, outputting angular velocity $\omega(t) \in \mathbb{R}^3$ and linear velocity $v(t) \in \mathbb{R}^3$ [S1].
4.  **Sampling:** At each iteration, 1024 rays are sampled, with 128 3D points per ray within a pre-defined depth range [S1].

**Numbered Steps:**

1.  **Initialize Networks:** Initialize $\phi_g$, $\phi_c$, and $\phi_v$. Normalize input timesteps to $[-1, 1]$ [S1].
2.  **Compute Camera Velocities:** For a given time $t$, compute angular velocity $\omega(t)$ and linear velocity $v(t)$:
    $$ \omega(t), v(t) = \phi_v(t) $$
    [S1]
3.  **Integrate Camera Motion:** To find the relative transformation from time $t$ to $t+l$ ($l>0$), use the Euler method with step size $\Delta t = l/U$ (where $U=10$) [S1]:
    $$ R_{t \to t+l} = \prod_{u=0}^{U-1} \psi(\omega(t + u\Delta t)\Delta t) $$
    $$ t_{t \to t+l} = \sum_{u=0}^{U-1} v(t + u\Delta t)\Delta t $$
    where $\psi(\cdot): \mathbb{R}^3 \to SO(3)$ converts rotation angles to a rotation matrix using right-matrix multiplication [S1].
4.  **Compute Transformation Matrix:** Define the transformation $P_{t_1 \to t_2}$ from time $t_1$ to $t_2$:
    $$ P_{t_1 \to t_2} = \begin{cases} B_{t_1 \to t_2} & \text{if } t_1 \le t_2 \\ (B_{t_2 \to t_1})^{-1} & \text{otherwise} \end{cases} $$
    where $B_{t_1 \to t_2} = \begin{bmatrix} R_{t_1 \to t_2} & t_{t_1 \to t_2} \\ 0 & 1 \end{bmatrix} \in \mathbb{R}^{4 \times 4}$ [S1].
5.  **Render Color:** For a ray $r$ at time $t$, compute the rendered color $\hat{C}_t(r)$ using volume rendering with the SDF $s(x,t) = \phi_g(x,t)$ and color $c(x,t) = \phi_c(x,d,t)$ [S1].
6.  **Compute Surface Point:** Calculate the expected surface point $\tilde{x}(r,t) \in \mathbb{R}^3$ for ray $r$ at time $t$:
    $$ \tilde{x}(r, t) = \sum_{i=1}^K L_i \sigma_i x_i(r, t) $$
    where $K$ is the number of samples, $L_i$ are weights, $\sigma_i$ are opacities, and $x_i$ are sample points [S1].
7.  **Calculate Losses:**
    *   **Rendering Loss ($L_{rgb}$):**
        $$ L_{rgb} = \frac{1}{|\Omega|} \sum_{r \in \Omega} \|\hat{C}_t(r) - C_t(r)\|^2 $$
        where $\Omega$ is the set of rays and $C_t(r)$ is the ground truth color [S1].
    *   **Eikonal Loss ($L_{eik}$):** Enforces the Eikonal equation for the SDF:
        $$ L_{eik} = \frac{1}{|X|} \sum_{x \in X} \left\| \frac{\partial s(x, t)}{\partial x} - 1 \right\|^2 $$
        where $X$ is the set of sampled 3D points [S1].
    *   **Flow Consistency Loss ($L_{flow}$):** Constrains SDF flow to match scene flow derived from camera motion:
        $$ L_{flow} = \frac{1}{|S|} \sum_{x \in S} \left\| \frac{\partial s(x,t)}{\partial t} + (\omega(t) \times x + v(t))^T n(x,t) \right\| $$
        where $S$ is the set of surface points and $n(x) = \nabla_x s(x,t)$ is the surface normal [S1].
    *   **Photometric Consistency Loss ($L_{photo}$):** Projects surface points to neighboring frames $t+n$ ($n \in \mathcal{N}$) and compares colors:
        $$ \tilde{x}'_{t \to t+n}(r, t) = P_{t \to t+n} \tilde{x}'(r, t) $$
        $$ L_{photo} = \frac{1}{|\mathcal{N}||\Omega|} \sum_{n,r} \| C_{t+n}[K \tilde{x}_{t \to t+n}] - C_t(r) \|_1 $$
        where $K$ is the intrinsic matrix and $[\cdot]$ denotes sampling [S1].
    *   **SDF Consistency Loss ($L_{sdf}$):** Encourages consistency between local SDF and the world frame SDF (where the world frame is the middle time step $t_w$):
        $$ L_{sdf} = \frac{1}{|X|} \sum_{x \in X} \| s(x, t) - s(x_{t \to t_w}, t_w) \| $$
        where $x_{t \to t_w}$ is point $x$ transformed to the world frame via $P_{t \to t_w}$ [S1].
8.  **Total Loss:**
    $$ L = L_{rgb} + \lambda_1 L_{eik} + \lambda_2 L_{flow} + \lambda_3 L_{photo} + \lambda_4 L_{sdf} $$
    Set $\lambda_4 = 0$ for the first 200 epochs to avoid noisy initial transformations, then gradually increase it [S1].
9.  **Joint Optimization:** Update $\phi_v, \phi_g, \phi_c$ using Adam optimizer with learning rate 0.001 [S1].
10. **Fine-tuning:** After joint optimization, fix the camera motion (derive poses $P_{t \to t_w}$), fix $t$ to the world time step $t_w$, drop $\lambda_2, \lambda_3, \lambda_4$, and fine-tune $\phi_g, \phi_c$ using all training frames for 5000 additional epochs with gradually decaying learning rate [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def integrate_camera_motion(phi_v, t_start, t_end, U=10):
    """
    Integrate camera velocities to get relative transformation.
    """
    l = t_end - t_start
    dt = l / U
    R = np.eye(3)
    t_vec = np.zeros(3)
    
    for u in range(U):
        t_curr = t_start + u * dt
        omega, v = phi_v(t_curr) # Returns (3,) and (3,)
        
        # Convert rotation vector to matrix (right-multiplication)
        # Assuming psi converts angle vector to rotation matrix
        delta_R = psi(omega * dt) 
        R = R @ delta_R
        
        t_vec += v * dt
        
    return R, t_vec

def compute_transformation(phi_v, t1, t2):
    """
    Compute P_{t1->t2} using integration from 0 or relative to each other.
    Simplified: integrate from min(t1,t2) to max(t1,t2) and invert if needed.
    In practice, integrate from a reference (e.g., 0) or use relative integration.
    Here we assume relative integration between t1 and t2.
    """
    if t1 <= t2:
        R, t_vec = integrate_camera_motion(phi_v, t1, t2)
        return np.array([[R[0,0], R[0,1], R[0,2], t_vec[0]],
                         [R[1,0], R[1,1], R[1,2], t_vec[1]],
                         [R[2,0], R[2,1], R[2,2], t_vec[2]],
                         [0, 0, 0, 1]])
    else:
        R, t_vec = integrate_camera_motion(phi_v, t2, t1)
        # Invert transformation
        R_inv = R.T
        t_inv = -R_inv @ t_vec
        return np.array([[R_inv[0,0], R_inv[0,1], R_inv[0,2], t_inv[0]],
                         [R_inv[1,0], R_inv[1,1], R_inv[1,2], t_inv[1]],
                         [R_inv[2,0], R_inv[2,1], R_inv[2,2], t_inv[2]],
                         [0, 0, 0, 1]])

def compute_loss(phi_g, phi_c, phi_v, rays, gt_colors, K, N_intervals, t_w, lambda_weights):
    """
    Compute total loss for a batch of rays.
    """
    # 1. Render Color and SDF
    # Assume sample_points, weights, sigmas are computed via volume rendering
    # using phi_g(x, t) and phi_c(x, d, t)
    rendered_colors, surface_points, sdfs, normals = render_volume(phi_g, phi_c, rays)
    
    # L_rgb
    l_rgb = np.mean((rendered_colors - gt_colors)**2)
    
    # L_eik
    # Compute gradient of SDF w.r.t x
    sdf_grads = compute_sdf_gradient(phi_g, surface_points)
    l_eik = np.mean((np.linalg.norm(sdf_grads, axis=1) - 1)**2)
    
    # L_flow
    # Compute time derivative of SDF and scene flow
    # dS/dt approx via finite difference or network output if available
    # Scene flow = omega x x + v
    t_curr = rays.time # Assuming rays have time attribute
    omega, v = phi_v(t_curr)
    scene_flow = np.cross(omega, surface_points) + v
    sdf_flow = compute_sdf_time_derivative(phi_g, surface_points, t_curr) # [G]: Implementation detail for dS/dt
    l_flow = np.mean(np.abs(sdf_flow + np.sum(scene_flow * normals, axis=1)))
    
    # L_photo
    l_photo = 0
    for n in N_intervals:
        t_neighbor = t_curr + n
        P = compute_transformation(phi_v, t_curr, t_neighbor)
        # Project surface point to neighbor frame
        homo_points = np.hstack([surface_points, np.ones((surface_points.shape[0], 1))])
        neighbor_points_homo = (P @ homo_points.T).T
        neighbor_points = neighbor_points_homo[:, :3]
        
        # Project to image plane
        neighbor_uv = K @ neighbor_points.T
        neighbor_uv = neighbor_uv[:2, :] / neighbor_uv[2, :]
        
        # Sample color from neighbor frame
        neighbor_colors = sample_image(K, neighbor_uv, gt_colors_neighbor) # [G]: Helper to sample GT color at UV
        
        l_photo += np.mean(np.abs(neighbor_colors - rendered_colors))
    l_photo /= len(N_intervals)
    
    # L_sdf
    # Transform points to world frame (t_w)
    P_to_world = compute_transformation(phi_v, t_curr, t_w)
    homo_points = np.hstack([surface_points, np.ones((surface_points.shape[0], 1))])
    world_points_homo = (P_to_world @ homo_points.T).T
    world_points = world_points_homo[:, :3]
    
    sdfs_world = phi_g(world_points, t_w)
    l_sdf = np.mean(np.abs(sdfs - sdfs_world))
    
    # Total Loss
    l_total = (l_rgb + 
               lambda_weights['eik'] * l_eik + 
               lambda_weights['flow'] * l_flow + 
               lambda_weights['photo'] * l_photo + 
               lambda_weights['sdf'] * l_sdf)
               
    return l_total, l_rgb, l_eik, l_flow, l_photo, l_sdf

def train_joint_optimization(phi_g, phi_c, phi_v, dataloader, epochs=200, lambda_weights_init=None):
    """
    Joint optimization phase.
    """
    optimizer = Adam([phi_g.parameters(), phi_c.parameters(), phi_v.parameters()], lr=0.001)
    
    for epoch in range(epochs):
        for rays, gt_colors in dataloader:
            optimizer.zero_grad()
            loss, *_ = compute_loss(phi_g, phi_c, phi_v, rays, gt_colors, ...)
            loss.backward()
            optimizer.step()
            
        # Adjust lambda_sdf after 200 epochs
        if epoch >= 200:
            lambda_weights_init['sdf'] = ... # Gradually increase

def train_fine_tuning(phi_g, phi_c, dataloader, t_w, N_intervals, K, gt_colors_all, epochs=5000):
    """
    Fine-tuning phase: Fix camera motion, update NeRF only.
    """
    optimizer = Adam([phi_g.parameters(), phi_c.parameters()], lr=0.001) # Decaying LR
    
    for epoch in range(epochs):
        for rays, gt_colors in dataloader:
            optimizer.zero_grad()
            # Recompute transformations based on fixed phi_v
            # Only compute L_rgb and L_eik (L_flow, L_photo, L_sdf dropped)
            # [G]: Implementation of L_rgb and L_eik as defined above
            l_rgb, l_eik = compute_rendering_and_eikonal_losses(phi_g, phi_c, rays, gt_colors)
            loss = l_rgb + lambda_weights_init['eik'] * l_eik
            loss.backward()
            optimizer.step()
```

**Parameters:**
*   $U = 10$: Number of sub-intervals between consecutive frames for integration [S1].
*   $t_w$: World time step, chosen as the middle frame in the training set [S1].
*   Learning rate: 0.001 for Adam during initial joint optimization [S1].
*   Fine-tuning epochs: 5000 additional epochs with gradually decaying learning rate [S1].
*   Sampling: 1024 rays and 128 3D points per ray within a pre-defined depth range [S1].
*   Input normalization: Timesteps normalized from -1 to 1 [S1].
*   $\lambda_4 = 0$ for the first 200 epochs [S1].

**Edge Cases and Pitfalls:**
*   **Noisy Initial Transformations:** Early in training, camera poses are noisy. Setting $\lambda_4 = 0$ for the first 200 epochs prevents the SDF consistency loss from hindering convergence [S1].
*   **Local Minima:** Using a single global NeRF with noisy initial transformations can get stuck in local minima. The time-dependent local representation avoids this [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On average, reduces depth square relative errors by at least 82% on Co3D and 78% on Scannet datasets compared to other methods [S1].
*   **Strengths:** Prior-free pipeline; does not rely on initial camera poses or intrinsics; handles complex camera motions (e.g., large rotations in Co3D) [S1].
*   **Limitations:** Requires monocular video; performance depends on the quality of the continuous motion modeling [S1].
*   **When to use:** When initial camera poses are unavailable or unreliable, and high-quality geometry reconstruction is needed from monocular video [S1].
*   **When not:** When high-quality initial poses are available and a simpler global NeRF optimization suffices.

## Background
*   **Origin:** The method was introduced in "Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video" [S1].
*   **History:** It builds on SDF-based NeRF models [29] and continuous camera motion modeling. It addresses limitations of prior works like NeRFmm [31] and NoPe-NeRF [2] which either require initial poses or use global NeRFs prone to local minima with noisy transformations [S1].
*   **Usage:** Evaluated on ScanNet (4 scenes) and Co3D (5 scenes) datasets for novel-view synthesis and depth evaluation [S1].

## Sources
[S1] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
