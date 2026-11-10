# Time-dependent NeRF with Continuous Camera Motion

Also called: COPE-NeRF training, Continuous Optimization of NeRF and poses, Joint Optimization Pipeline, Joint Optimization of Neural Radiance Fields and Continuous Camera Motion

## Overview
This method jointly optimizes a time-dependent Neural Radiance Field (NeRF) and continuous camera motion from monocular video, representing scene geometry as local Signed Distance Fields (SDFs) and appearance at each time step [S1]. Unlike prior approaches that train a single global NeRF mapped via noisy initial transformations, this method uses local time-dependent representations constrained by consistency losses to avoid local minima and accurately reconstruct full scene geometry [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A monocular video sequence (RGB images), camera intrinsics $K$, and timestamps for each frame.
- **Outputs:** A time-dependent NeRF model ($\phi_g, \phi_c$) representing scene geometry and appearance, and continuous camera motion parameters ($\omega(t), v(t)$) from which camera poses for all frames are derived [S1].

**Data Structures:**
1.  **Motion Network ($\phi_v$):** An MLP mapping time $t$ to angular velocity $\omega(t) \in \mathbb{R}^3$ and linear velocity $v(t) \in \mathbb{R}^3$ [S1].
2.  **Geometry Network ($\phi_g$):** An MLP mapping 3D point $x \in \mathbb{R}^3$ and time $t$ to SDF value $s(x,t) \in \mathbb{R}$ [S1].
3.  **Appearance Network ($\phi_c$):** An MLP mapping 3D point $x \in \mathbb{R}^3$, viewing direction $d \in \mathbb{R}^2$ (in camera coordinates), and time $t$ to color $c(x,t) \in \mathbb{R}^3$ [S1].
4.  **Time Normalization:** Input timesteps are normalized to the range $[-1, 1]$ for all networks [S1].

**Numbered Steps:**

1.  **Initialize Networks:** Initialize $\phi_v$, $\phi_g$, and $\phi_c$. Set learning rate to 0.001 for Adam optimizer [S1].
2.  **Joint Optimization (Early Stage):**
    -   Sample rays and 3D points. For each iteration, sample 1024 rays and 128 3D points per ray within a pre-defined depth range [S1].
    -   Compute rendered color $\hat{C}_t(r)$ using volume rendering (see formulas below) [S1].
    -   Compute loss components:
        -   Rendering loss $L_{rgb}$ [S1].
        -   Eikonal loss $L_{eik}$ [S1].
        -   Flow consistency loss $L_{flow}$ [S1].
        -   Photometric consistency loss $L_{photo}$ [S1].
        -   Geometry consistency loss $L_{sdf}$ [S1].
    -   Compute total loss $L = L_{rgb} + \lambda_1 L_{eik} + \lambda_2 L_{flow} + \lambda_3 L_{photo} + \lambda_4 L_{sdf}$ [S1].
    -   **Parameter Note:** Set $\lambda_4 = 0$ for the first 200 epochs to avoid noisy initial transformations, then gradually increase it [S1].
    -   Update $\phi_v, \phi_g, \phi_c$ using gradient descent [S1].
3.  **Derive Camera Poses:** After joint optimization, compute relative camera transformations between any two time steps $t_1$ and $t_2$ using the integrated velocities (see formulas below) [S1].
4.  **Fine-Tuning (Late Stage):**
    -   Fix the world time step $t_w$ to the middle frame of the training set [S1].
    -   Fix camera poses $P_{t \to t_w}$ derived from the learned motions [S1].
    -   Drop weights $\lambda_2, \lambda_3, \lambda_4$ (set to 0) [S1].
    -   Update only $\phi_g$ and $\phi_c$ using all available training frames to reconstruct the full scene geometry in the world coordinate system [S1].
    -   Continue for 5000 additional epochs with gradually decaying learning rate [S1].

**Formulas:**

*Camera Motion Integration:*
The continuous camera motion is defined by angular velocity $\omega(t)$ and linear velocity $v(t)$. The relationship between camera rotation $R \in SO(3)$ and translation $t \in \mathbb{R}^3$ is:
$$ \frac{dR}{dt} = [\omega(t)]_\times R, \quad \frac{dt}{dt} = v(t) $$
where $[\omega(t)]_\times \in \mathfrak{so}(3)$ is the skew-symmetric matrix of $\omega(t)$ [S1].

Relative rotation $R_{t \to t+l}$ and translation $t_{t \to t+l}$ from time $t$ to $t+l$ ($l > 0$) are computed using the Euler method with step size $\Delta t = l/U$ and $U=10$ sub-intervals [S1]:
$$ R_{t \to t+l} = \prod_{u=0}^{U-1} \psi(\omega(t + u\Delta t)\Delta t) $$
$$ t_{t \to t+l} = \sum_{u=0}^{U-1} v(t + u\Delta t)\Delta t $$
where $\psi(\cdot): \mathbb{R}^3 \to SO(3)$ converts rotation angles to a rotation matrix, and right-matrix multiplication is applied [S1].

The transformation matrix $P_{t_1 \to t_2}$ transforming coordinates from time $t_1$ to $t_2$ is:
$$ P_{t_1 \to t_2} = \begin{cases} B_{t_1 \to t_2} & \text{if } t_1 \le t_2 \\ (B_{t_2 \to t_1})^{-1} & \text{otherwise} \end{cases} $$
where $B_{t_1 \to t_2} = \begin{bmatrix} R_{t_1 \to t_2} & t_{t_1 \to t_2} \\ 0 & 1 \end{bmatrix} \in \mathbb{R}^{4 \times 4}$ [S1].

*NeRF Representation:*
The time-dependent NeRF predicts SDF $s(x,t)$ and color $c(x,t)$ for a point $x$ in the local camera coordinate system at time $t$:
$$ s(x, t) = \phi_g(x, t) $$
$$ c(x, t) = \phi_c(x, d, t) $$
where $d \in \mathbb{R}^2$ is the viewing direction in the camera coordinate system [S1].

*Surface Point Computation:*
The surface point $\tilde{x}(r, t) \in \mathbb{R}^3$ at time $t$ of ray $r$ is computed via volume rendering:
$$ \tilde{x}(r, t) = \sum_{i=1}^K L_i \sigma_i x_i(r, t) $$
where $K$ is the number of samples, $L_i$ is the transmittance, $\sigma_i$ is the density, and $x_i$ are the sampled points [S1].

*Loss Functions:*
1.  **Rendering Loss:**
    $$ L_{rgb} = \frac{1}{|\Omega|} \sum_{r \in \Omega} \|\hat{C}_t(r) - C_t(r)\|^2 $$
    where $\Omega$ is the set of rays, $\hat{C}_t(r)$ is the rendered color, and $C_t(r)$ is the ground truth [S1].

2.  **Eikonal Loss:**
    $$ L_{eik} = \frac{1}{|X|} \sum_{x \in X} \left\| \frac{\partial s(x, t)}{\partial x} - 1 \right\|^2 $$
    where $X$ is the set of sampled 3D points, enforcing the Eikonal equation for SDF [S1].

3.  **Flow Consistency Loss:**
    $$ L_{flow} = \frac{1}{|S|} \sum_{x \in S} \left\| \frac{\partial s(x,t)}{\partial t} + (\omega(t) \times x + v(t))^T n(x,t) \right\| $$
    where $S$ is the set of surface points, and $n(x) = \frac{\partial s(x,t)}{\partial x} \in \mathbb{R}^3$ is the surface normal [S1].

4.  **Photometric Consistency Loss:**
    Project surface point to neighboring frame $t+n$:
    $$ \tilde{x}'_{t \to t+n}(r, t) = P_{t \to t+n} \tilde{x}'(r, t) $$
    where $\tilde{x}'$ denotes homogeneous coordinates [S1].
    $$ L_{photo} = \frac{1}{|N||\Omega|} \sum_{n,r} \| C_{t+n}[K \tilde{x}_{t \to t+n}] - C_t(r) \|_1 $$
    where $n \in N$ (hyper-parameter intervals), $K$ is the intrinsic matrix, and $[\cdot]$ denotes sampling [S1].

5.  **Geometry Consistency Loss:**
    $$ L_{sdf} = \frac{1}{|X|} \sum_{x \in X} \| s(x, t) - s(x_{t \to t_w}, t_w) \| $$
    where $t_w$ is the middle frame (world frame), and $x_{t \to t_w}$ is point $x$ transformed to the world frame via $P_{t \to t_w}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_skew_matrix(w):
    """Converts angular velocity vector w to skew-symmetric matrix."""
    return np.array([
        [0, -w[2], w[1]],
        [w[2], 0, -w[0]],
        [-w[1], w[0], 0]
    ])

def exp_so3(w):
    """Converts rotation vector w to rotation matrix using Rodrigues' formula approximation or similar.
    Note: The source uses psi(w * dt) to convert rotation angles to rotation matrix.
    For implementation, this typically implies exp([w]_x) or similar SO(3) exponential map.
    Here we assume a standard SO(3) exponential map implementation exists."""
    # Placeholder for standard SO(3) exponential map from rotation vector
    # In practice, use scipy.spatial.transform.Rotation or similar
    pass

def integrate_motion(phi_v, t_start, t_end, U=10):
    """Integrates camera motion from t_start to t_end."""
    l = t_end - t_start
    dt = l / U
    R = np.eye(3)
    t_vec = np.zeros(3)
    
    for u in range(U):
        t_curr = t_start + u * dt
        omega, v = phi_v(t_curr) # Returns (omega, v)
        
        # Rotation update: Right-matrix multiplication
        # R_new = R_old * psi(omega * dt)
        rot_vec = omega * dt
        R_step = exp_so3(rot_vec)
        R = R @ R_step
        
        # Translation update
        t_vec += v * dt
        
    return R, t_vec

def compute_transformation_matrix(R, t_vec):
    """Constructs 4x4 transformation matrix."""
    B = np.eye(4)
    B[:3, :3] = R
    B[:3, 3] = t_vec
    return B

def get_pose_matrix(phi_v, t1, t2):
    """Gets transformation matrix P_{t1->t2}."""
    if t1 <= t2:
        R, t_vec = integrate_motion(phi_v, t1, t2)
        B = compute_transformation_matrix(R, t_vec)
        return B
    else:
        R, t_vec = integrate_motion(phi_v, t2, t1)
        B = compute_transformation_matrix(R, t_vec)
        return np.linalg.inv(B)

def volume_render(phi_g, phi_c, ray_o, ray_d, t, num_samples=128):
    """Performs volume rendering for a single ray at time t."""
    # Sample points along ray within depth range
    # x_i = ray_o + h_i * ray_d
    # Compute sigma and c for each point
    # Compute transmittance L_i
    # Return rendered color and surface point estimate
    pass

def train_joint_optimization(phi_v, phi_g, phi_c, dataset, epochs=200):
    """Early stage joint optimization."""
    optimizer = Adam([phi_v.parameters(), phi_g.parameters(), phi_c.parameters()], lr=0.001)
    lambda_4 = 0.0
    
    for epoch in range(epochs):
        if epoch > 200:
            lambda_4 = lambda_4 + 0.001 # Gradually increase
            
        for batch in dataset:
            rays, gt_colors = batch
            
            # Sample 3D points for SDF
            points = sample_3d_points(rays, num_points=128)
            
            # Forward pass
            omega, v = phi_v(t) # t is time for current frame
            s = phi_g(points, t)
            c = phi_c(points, directions, t)
            
            # Render
            rendered_colors = volume_render(phi_g, phi_c, rays.o, rays.d, t)
            
            # Compute Losses
            L_rgb = mse(rendered_colors, gt_colors)
            L_eik = eikonal_loss(s, points)
            L_flow = flow_loss(s, t, omega, v, points)
            L_photo = photometric_loss(rays, t, omega, v, phi_g, phi_c, K)
            L_sdf = sdf_consistency_loss(s, t, phi_g, phi_c, t_w, points)
            
            L = L_rgb + lambda_1 * L_eik + lambda_2 * L_flow + lambda_3 * L_photo + lambda_4 * L_sdf
            
            optimizer.zero_grad()
            L.backward()
            optimizer.step()

def train_fine_tuning(phi_g, phi_c, dataset, t_w, poses, epochs=5000):
    """Late stage fine-tuning."""
    optimizer = Adam([phi_g.parameters(), phi_c.parameters()], lr=0.001)
    
    for epoch in range(epochs):
        for batch in dataset:
            rays, gt_colors = batch
            
            # Transform rays to world coordinates using fixed poses
            # Or render in local coordinates and transform points?
            # Source says: "fix camera poses... update only (phi_g, phi_c) using all training frames"
            # This implies training the global NeRF at t_w using transformed data or consistent local data.
            
            # Forward pass at t_w
            s = phi_g(points, t_w)
            c = phi_c(points, directions, t_w)
            
            rendered_colors = volume_render(phi_g, phi_c, rays.o, rays.d, t_w)
            
            L_rgb = mse(rendered_colors, gt_colors)
            L_eik = eikonal_loss(s, points)
            
            # Lambda 2, 3, 4 are 0
            L = L_rgb + lambda_1 * L_eik
            
            optimizer.zero_grad()
            L.backward()
            optimizer.step()
```

**Parameters:**
-   $U = 10$: Number of sub-intervals between two consecutive frames [S1].
-   $t_w$: World time step, chosen as the middle frame in the training set [S1].
-   Learning rate: 0.001 for Adam optimizer during initial training [S1].
-   Fine-tuning epochs: 5000 additional epochs with gradually decaying learning rate [S1].
-   Sampling: 1024 rays and 128 3D points per ray within a pre-defined depth range at each training iteration [S1].
-   Input normalization: Timesteps normalized from -1 to 1 for all models [S1].

**Edge Cases and Pitfalls:**
-   **Noisy Initial Transformations:** In the early training stage, inferred camera-to-world transformations are noisy. Setting $\lambda_4 = 0$ for the first 200 epochs prevents the geometry consistency loss from destabilizing the optimization [S1].
-   **Local Minima:** Using a single global NeRF with noisy initial transformations can cause the optimization to get stuck in local minima. The time-dependent local representation avoids this [S1].

## Operating characteristics
-   **Time and Memory Complexity:** Not given in the sources.
-   **Measured Performance:** On average, reduces depth square relative errors by at least 82% on Co3D and 78% on Scannet datasets compared to other methods [S1].
-   **Scaling:** Not given in the sources.
-   **Strengths:** Prior-free pipeline; jointly optimizes camera motion and scene geometry; does not rely on prior knowledge about camera motion or scene; produces correct scene geometry given only RGB images [S1].
-   **Limitations:** Not given in the sources.
-   **When to use:** When accurate camera poses and full scene geometry reconstruction are required from monocular video, especially with complex camera motions [S1].
-   **When not:** Not given in the sources.

## Background
-   **Origin:** The method was introduced in "Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video" [S1].
-   **History and Development:** It builds upon SDF-based NeRF models [S1] and leverages the linear relationship between scene flow and SDF flow from [18] [S1]. It addresses limitations of prior works like NeRFmm [31] and NoPe-NeRF [2] which either require additional geometric information or struggle with noisy initial transformations [S1].
-   **Usage:** Evaluated on ScanNet (4 scenes) and Co3D (5 scenes) datasets for novel-view synthesis and depth evaluation [S1].

## Sources
[S1] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
