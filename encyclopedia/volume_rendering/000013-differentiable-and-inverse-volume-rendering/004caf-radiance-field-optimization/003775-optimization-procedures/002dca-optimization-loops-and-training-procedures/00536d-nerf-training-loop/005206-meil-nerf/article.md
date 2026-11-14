# MEIL-NeRF

Also called: Memory-Efficient Incremental Learning of NeRF

## Overview
MEIL-NeRF is a memory-efficient incremental learning algorithm for Neural Radiance Fields (NeRF) that prevents catastrophic forgetting without increasing memory usage by treating the NeRF network itself as a memory storage for pixel RGB values [S1]. Unlike replay-based methods that suffer from monotonically increasing memory due to storing past data, MEIL-NeRF maintains constant memory consumption by using a Ray Generator Network (RGN) to retrieve past task information from the NeRF network itself, rather than storing raw data or exemplars [S1]. It builds on [NeRF Training Loop](970-nerf-training-loop.md) by introducing a self-distillation mechanism guided by generated rays.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Previous task parameters $\theta_{T-1}$, current task data (rays $r_c$ and colors $C_c$), camera intrinsics (focal length $f$, image dimensions $W, H$), and hyperparameters ($T, N, m_c, \lambda_p$).
*   **Outputs:** Updated NeRF parameters $\theta_T$ and updated RGN parameters.

**Data Structures:**
*   **NeRF Network ($F$):** An MLP mapping 3D coordinates and viewing directions to density $\sigma$ and color $c$. Parameters $\theta$.
*   **Ray Generator Network (RGN, $f_{RGN}$):** A small MLP mapping a scalar $x \in [0, 1]$ to a ray origin $r^*_o$ and direction $r^*_d$.
*   **Frozen Parameters ($\theta^*_{T-1}$):** A copy of $\theta_{T-1}$ used for querying past data.

**Volume Rendering:**
The estimated pixel color $\hat{C}$ for a ray $r$ is computed via standard volume rendering [S1]:
$$ \hat{C} = \sum_{i=1}^{P} \left( \prod_{j=1}^{i-1} (1 - \alpha_j) \right) \alpha_i c_i $$
where $\alpha_i = 1 - \exp(-\sigma_i(z_{i+1} - z_i))$, $P$ is the number of sampled points, $c_i$ is color, $\sigma_i$ is density, and $z_i$ is distance from the camera. We denote the forward mapping as $\hat{C} = F(r; \theta)$.

**Algorithm Steps:**
1.  **Initialization:** For task $T$, copy parameters from the previous task: $\theta^*_{T-1} = \theta_{T-1}$. Freeze $\theta^*_{T-1}$.
2.  **Iteration Loop:** For each training iteration of task $T$:
    a. Sample current task data $(r_c, C_c)$ with batch size $m_c$.
    b. If $T > 1$:
        i. Generate past principal rays $r^*_p$ using the RGN: $(r^*_o, r^*_d) = f_{RGN}(x)$, where $x$ is sampled or spaced.
        ii. Generate non-principal rays $r_p$ around the principal rays using camera intrinsics (see Formula 7 below).
        iii. Query the frozen network to obtain past pseudo-ground truth: $C_p = F(r_p; \theta^*_{T-1})$.
    c. Compute current predictions $\hat{C}_c = F(r_c; \theta_T)$ and past predictions $\hat{C}_p = F(r_p; \theta_T)$.
    d. Update $\theta_T$ by minimizing the loss $L$ (see Formula 4 below).
3.  **RGN Update:** After training for task $T$ ends, update the RGN using current principal rays and past principal rays retrieved from the previous RGN state to map $T \cdot N$ equally spaced numbers to rays using MSE loss (see Formula 8 below).

**Formulas:**

1.  **Non-principal Ray Generation:**
    Centered around the generated principal ray $(r^*_o, r^*_d)$, non-principal rays $(r_o, r_d)$ are created with the same origin $r_o = r^*_o$ [S1]:
    $$ r_d = \frac{f \cdot r^*_d + s \cdot \cos(u_\theta) \cdot p_1 + s \cdot \sin(u_\theta) \cdot p_2}{|f \cdot r^*_d + s \cdot \cos(u_\theta) \cdot p_1 + s \cdot \sin(u_\theta) \cdot p_2|} $$
    where $p_1, p_2$ are unit vectors perpendicular to the principal ray (obtained via Gram-Schmidt), $f$ is focal length, $s \sim U[0, \sqrt{W^2+H^2}/2]$, and $u_\theta \sim U[0, 2\pi]$ [S1].

2.  **Loss Function:**
    $$ L = \frac{1}{m_c} \sum_{c} (\hat{C}_c - C_c)^2 + \lambda_p \frac{1}{m_p} \sum_{p} \rho(\hat{C}_p - C_p) $$
    where $\rho(x) = \sqrt{x^2 + \epsilon^2}$ is the Charbonnier penalty function [S1]. $\lambda_p$ controls the trade-off. $m_c$ is the batch size of current rays. $m_p$ is the batch size of past rays. Default $m_c = 4096$. Default $m_p = m_c / 2$ [S1].

3.  **RGN Training Loss:**
    The RGN is trained to map $T \cdot N$ equally spaced numbers $x_T \in [0, 1]$ to principal rays $(r^*_o, r^*_d)_1:T$ [S1]:
    $$ L_{RGN} = \| (r^*_o, r^*_d)_1:T - f_{RGN}(x_T) \|^2 $$
    where $x_T$ are spaced values covering the range $[0, 1]$ corresponding to all seen tasks [S1].

**Parameters:**
*   $T$: Number of seen tasks.
*   $N$: Number of images per task. Default $N=5$ [S1].
*   $m_c$: Batch size of current rays. Default $4096$ [S1].
*   $m_p$: Batch size of past rays. Default $m_c/2$ [S1].
*   $\lambda_p$: Trade-off hyperparameter. Scheduled based on task progress rate $r$ (see Subtopics).
*   Iterations per task: $3000 \times N$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def gram_schmidt(v):
    """Returns two unit vectors perpendicular to v."""
    # Simplified implementation for 3D vector v
    # In practice, use a robust cross-product method
    if abs(v[0]) < 0.9:
        u1 = np.cross(v, [1, 0, 0])
    else:
        u1 = np.cross(v, [0, 1, 0])
    u1 = u1 / np.linalg.norm(u1)
    u2 = np.cross(v, u1)
    u2 = u2 / np.linalg.norm(u2)
    return u1, u2

def generate_non_principal_ray(principal_dir, focal_length, width, height, rng):
    """Generates a non-principal ray direction around the principal direction."""
    p1, p2 = gram_schmidt(principal_dir)
    s = rng.uniform(0, math.sqrt(width**2 + height**2) / 2)
    u_theta = rng.uniform(0, 2 * math.pi)
    
    raw_dir = focal_length * principal_dir + \
              s * math.cos(u_theta) * p1 + \
              s * math.sin(u_theta) * p2
              
    return raw_dir / np.linalg.norm(raw_dir)

def charbonnier_loss(x, eps=1e-3):
    return math.sqrt(x**2 + eps**2)

def train_meil_nerf(task_idx, prev_theta, current_data, camera_intrinsics, 
                    rgn_model, prev_rgn_model, num_images_per_task=5, 
                    iterations_per_task=15000, mc=4096, mp=2048):
    """
    task_idx: Current task T (1-based)
    prev_theta: Parameters from task T-1
    current_data: List of (rays, colors) for current task
    camera_intrinsics: dict with 'f', 'W', 'H'
    rgn_model: Current RGN model
    prev_rgn_model: RGN model from previous task (for self-distillation update)
    """
    # 1. Freeze previous parameters
    frozen_theta = copy_parameters(prev_theta)
    
    # 2. Training Loop
    for iteration in range(iterations_per_task):
        # Sample current data
        r_c, C_c = sample_batch(current_data, mc)
        
        # Compute current loss (L2)
        C_c_pred = volume_render(r_c, frozen_theta) # Note: Use theta_T being updated, not frozen
        # Actually, we update theta_T, so we need the active theta
        # Let's assume theta is the active parameter set
        theta = copy_parameters(prev_theta) # Start with prev
        
        # If T > 1, generate past rays
        if task_idx > 1:
            # Generate principal rays using RGN
            # x values for past tasks
            num_past_tasks = task_idx - 1
            x_past = np.linspace(0, 1, num_past_tasks * num_images_per_task)
            
            # Get principal rays from RGN
            # RGN outputs (origin, direction)
            # We only need direction for the ray generation formula, 
            # but origin is needed for the ray. 
            # The formula assumes origin is same as principal ray origin.
            principal_origins, principal_dirs = rgn_model(x_past)
            
            # Generate non-principal rays
            r_p_list = []
            for i in range(mp):
                # Pick a random principal ray index
                idx = np.random.randint(0, len(principal_dirs))
                p_dir = principal_dirs[idx]
                p_orig = principal_origins[idx]
                
                # Generate variation
                r_d = generate_non_principal_ray(p_dir, camera_intrinsics['f'], 
                                                 camera_intrinsics['W'], 
                                                 camera_intrinsics['H'], 
                                                 np.random.default_rng())
                r_p_list.append((p_orig, r_d))
            
            r_p, C_p_origins = zip(*r_p_list) # r_p contains origins and dirs
            # Convert to ray format expected by volume_render
            r_p = format_rays(r_p)
            
            # Query frozen network for past pseudo-ground truth
            C_p = volume_render(r_p, frozen_theta)
            
            # Compute past loss (Charbonnier)
            C_p_pred = volume_render(r_p, theta)
            
            # Calculate lambda_p based on progress rate r
            r = iteration / iterations_per_task
            lambda_p = schedule_lambda_p(r)
            
            # Total Loss
            loss_c = mean_squared_error(C_c_pred, C_c)
            loss_p = mean_charbonnier(C_p_pred, C_p)
            loss = loss_c + lambda_p * loss_p
            
        else:
            # Task 1: Only current loss
            C_c_pred = volume_render(r_c, theta)
            loss = mean_squared_error(C_c_pred, C_c)
            lambda_p = 0
            
        # Update theta using gradient descent on loss
        theta = update_parameters(theta, loss)
        
    # 3. Update RGN
    # Retrieve past principal rays from previous RGN
    if task_idx > 1:
        num_past = (task_idx - 1) * num_images_per_task
        x_past = np.linspace(0, 1, num_past)
        past_principal_rays = prev_rgn_model(x_past)
    else:
        past_principal_rays = []
        
    # Current principal rays (Ground Truth)
    # Assuming current_data contains camera poses to generate principal rays
    current_principal_rays = generate_principal_rays(current_data, num_images_per_task)
    
    all_principal_rays = past_principal_rays + current_principal_rays
    num_total = len(all_principal_rays)
    x_all = np.linspace(0, 1, num_total)
    
    # Train RGN to map x_all to all_principal_rays
    rgn_loss = mean_squared_error(rgn_model(x_all), all_principal_rays)
    rgn_model = update_parameters(rgn_model, rgn_loss)
    
    return theta, rgn_model

def schedule_lambda_p(r):
    """Schedule 2 from sources."""
    return math.cos(math.pi * (1 + r))
```

## Subtopics

### λp Scheduling Strategy
A learning curriculum that schedules the hyperparameter $\lambda_p$ according to the task progress rate $r$ to shift focus from learning a new task to remembering past tasks [S1]. The progress rate $r$ ranges from 0.0 to 1.0 [S1]. Several schedules were tested:
*   Schedule 1: $\lambda_p = \cos(\frac{\pi}{2}(1-r))$
*   Schedule 2: $\lambda_p = \cos(\pi(1+r))$
*   Schedule 3: $\lambda_p = \cos(\frac{\pi}{2}(1+3r))$
*   Schedule 4: $\lambda_p = 10^{\frac{3}{7}\cos(\pi(1+\frac{2}{7}r))}$
*   Schedule 5: $\lambda_p = 10^{\frac{1}{7}\cos(\pi(1+\frac{2}{7}r))}$ [S1]
Schedule 2 demonstrates slightly better results among Schedules 1-3 and is selected for implementation [S1].

### Loss Function Selection for Past Tasks
Selection of L1 or Charbonnier loss over L2 for remembering past tasks to preserve knowledge and improve edge-preserving capability [S1]. L1 and Charbonnier losses provide better edge-preserving capability and reduce errors from the iterative self-distillation process compared to L2 loss [S1]. On TUM-RGBD/Seq0, Charbonnier + $\lambda_p$ scheduling achieves PSNR 23.70 / MS-SSIM 0.884, outperforming L2 (23.85 / 0.873) in MS-SSIM [S1].

## Operating characteristics
*   **Memory:** Constant memory consumption regardless of the number of tasks, as it does not store raw past data or exemplars [S1].
*   **Time Complexity:** Adds overhead for RGN generation and querying the frozen network for past rays. Iterations per task are $3000 \times N$ [S1].
*   **Performance:** Competitive performance compared to joint training and replay-based methods like iMAP* [S1]. Outperforms NeRF-Incre (standard incremental) significantly [S1].
*   **Strengths:** Prevents catastrophic forgetting without memory scalability issues.
*   **Limitations:** Requires careful scheduling of $\lambda_p$ and tuning of batch sizes ($m_c, m_p$) [S1].

## Background
MEIL-NeRF was proposed to address catastrophic forgetting in NeRF under incremental learning scenarios, where only recent viewpoints are accessible [S1]. It draws inspiration from the NeRF network's ability to store scene information, using it as a memory storage for pixel RGB values [S1]. The method was compared against NeRF-Incre, NeRF-Joint, EWC, PackNet, iMAP, and iMAP* on datasets such as Tanks and Temples, Replica, and TUM-RGBD [S1].

## Sources
[S1] MEIL-NeRF: Memory-Efficient Incremental Learning of Neural Radiance Fields, https://arxiv.org/abs/2212.08328v3, fetched 2026-10-26
