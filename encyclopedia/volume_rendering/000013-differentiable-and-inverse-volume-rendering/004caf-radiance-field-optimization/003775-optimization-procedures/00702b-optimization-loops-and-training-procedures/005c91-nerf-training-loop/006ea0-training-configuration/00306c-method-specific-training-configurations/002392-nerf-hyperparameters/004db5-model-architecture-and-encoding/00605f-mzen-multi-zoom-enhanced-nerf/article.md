# MZEN (Multi-zoom Enhanced NeRF)

Also called: Multi-zoom Enhanced NeRF

## Overview
MZEN is a framework for 3-D reconstruction from images with unknown camera poses that explicitly models optical zoom as a learnable intrinsic parameter and uses cross-zoom pose registration to align non-overlapping zoomed-in and zoomed-out views [S1]. It solves the problem of stable training across multiple zoom levels without requiring high-resolution images or external pose initialization [S1]. Unlike [CamP](15539-ue4-nerf.md) which perturbs focal length per view, or [Mip-NeRF](15532-rip-nerf.md) and [Zip-NeRF](15532-rip-nerf.md) which rely on image pyramids, MZEN keeps every frame at native resolution and assumes a single base focal length shared by all views, scaling it via a per-view zoom parameter [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A set of $N$ images $\mathcal{I} = \{I_k\}_{k=1}^N$.
*   The set is partitioned into $N_{out}$ zoomed-out images (indices $i=1 \dots N_{out}$) and $N_{in}$ zoomed-in images (indices $j=N_{out}+1 \dots N$) [S1].
*   Initial rough estimates of zoom parameters $\xi_k$ (e.g., from camera dial readings or apparent size comparison) [S1].
*   Base focal length $f = (f_x, f_y)$ and principal point $c = (c_x, c_y)$ [S1].

**Outputs:**
*   Optimized NeRF parameters $\Theta$ (radiance field MLP weights) [S1].
*   Optimized camera poses $\Pi = \{R_k, t_k, \xi_k\}_{k=1}^N$ (rotation, translation, zoom) [S1].

### Data Structures
The camera pose for image $k$ is defined as $\Pi_k = (R_k, t_k, f, c, \xi_k)$ [S1].
*   $R_k \in SO(3)$: Rotation matrix.
*   $t_k \in \mathbb{R}^3$: Translation vector.
*   $f \in \mathbb{R}^2$: Shared base focal length $(f_x, f_y)$ [S1].
*   $c \in \mathbb{R}^2$: Principal point $(c_x, c_y)$ [S1].
*   $\xi_k \in \mathbb{R}_{\ge 1}$: Learnable zoom parameter for view $k$ [S1].

The effective intrinsic matrix is:
$$ K_{eff}^k = \text{diag}(\xi_k f_x, \xi_k f_y, 1) $$
[S1]

### Steps
The training proceeds in three distinct phases to ensure stability [S1].

**Phase A: Global Pose Bootstrap**
1.  Initialize zoom parameters for the $N_{out}$ most zoomed-out images: $\xi_i = 1$ for $i=1 \dots N_{out}$ [S1].
2.  Randomly initialize NeRF parameters $\Theta^{(A)}$, rotations $R_i$, translations $t_i$, and focal lengths $f^{(A)}$ [S1].
3.  Withhold all zoomed-in images ($j > N_{out}$) from the loss calculation [S1].
4.  Minimize the coarse loss $L_{coarse}$ over the $N_{out}$ images to obtain coarse NeRF parameters and globally consistent poses [S1].

**Phase B: Cross-Zoom Pose Registration**
1.  Freeze NeRF parameters at $\Theta^{(A)}$ [S1].
2.  For each zoomed-in image $I_j$ ($j > N_{out}$):
    a.  Find the best matching zoomed-out image $g^*(j)$ from the set of $N_{out}$ images by minimizing the RGB mean-squared error between $I_j$ and a surrogate view $\tilde{I}_{g \to j}$ [S1].
    b.  The surrogate view $\tilde{I}_{g \to j}$ is created by extracting the central $1/\xi_j$ crop from the zoomed-out image $I_g$ and bilinearly resizing it to the native resolution of $I_j$ [S1].
    c.  Initialize the pose $\Pi_j$ for image $j$ using the parameters $(R_g, t_g, f, c)$ from the matched zoomed-out image $g^*(j)$ and the zoom parameter $\xi_j$ [S1]. This is "pose priming" [S1].
3.  Optimize only the rotation $R_j$, translation $t_j$, and zoom parameter $\xi_j$ for each zoomed-in image $j$ by minimizing $L_{pose}$ [S1]. Intrinsics $f, c$ remain fixed from the surrogate [S1].
4.  The output is the provisional full pose set $\Pi^{(B)} = \{\Pi_i^{(A)}\}_{i=1}^{N_{out}} \cup \{\Pi_j^{(B)}\}_{j=N_{out}+1}^{N}$ [S1].

**Phase C: Joint Refinement**
1.  Initialize $\Theta^{(C)} = \Theta^{(A)}$ [S1].
2.  Unfreeze NeRF parameters and all camera pose parameters (including all $\xi_k$) [S1].
3.  Optimize $\Theta^{(C)}$ and $\Pi^{(B)}$ jointly for all $N$ images by minimizing $L_{fine}$ [S1].
4.  This step sharpens fine details and yields pixel-accurate poses [S1].

### Formulas
**Rendering:**
The rendered image $\hat{I}_k$ for camera $k$ is computed using the differentiable volume renderer $F_\Theta$:
$$ \hat{I}_k = F_\Theta(\Theta, \Pi_k) $$
[S1]

**Loss Functions:**
*   **Phase A Loss:**
    $$ L_{coarse} = \sum_{i=1}^{N_{out}} \| F_{\Theta^{(A)}}(\Theta^{(A)}, \Pi^{(A)}) - I_i \|_2^2 $$
    [S1]
*   **Phase B Loss:**
    $$ L_{pose} = \sum_{j=N_{out}+1}^{N} \| F_{\Theta^{(A)}}(\Theta^{(A)}, \Pi^{(B)}) - I_j \|_2^2 $$
    [S1]
*   **Phase C Loss:**
    $$ L_{fine} = \sum_{k=1}^{N} \| F_{\Theta^{(C)}}(\Theta^{(C)}, \Pi^{(B)}) - I_k \|_2^2 $$
    [S1]
*   **General Photometric Loss:**
    $$ L(\Theta, \Pi) = \sum_{k=1}^{N} \| F_\Theta(\Theta, \Pi_k) - I_k \|_2^2 $$
    [S1]

**Convergence Analysis (Phase B):**
Under the Polyak-Łojasiewicz (PL) condition and L-smoothness, full-batch gradient descent with step size $\eta$ converges linearly [S1]:
$$ e^{(t+1)} \leq (1 - \eta\mu) e^{(t)} $$
where $e^{(t)} = L_j(\Pi_j^{(t)}) - L^*_j$ is the excess loss, $\mu > 0$ is the PL constant, and $\eta \in (0, 1/L]$ where $L$ is the Lipschitz constant of the gradients [S1].

### Parameters
*   **Zoom parameter $\xi_k$:** Learnable intrinsic parameter. $\xi_k = 1$ for zoomed-out frames, $\xi_k > 1$ for zoomed-in frames [S1].
*   **Step size $\eta$:** Must satisfy $\eta \in (0, 1/L]$ [S1].
*   **PL constant $\mu$:** Positive constant defining local strong convexity/PL condition [S1].
*   **Lipschitz constant $L$:** Positive constant such that $\|\nabla L_j(a) - \nabla L_j(b)\|_2 \leq L\|a - b\|_2$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mzen_training(images, initial_zoom_estimates, base_focal_length, principal_point):
    """
    images: List of N images.
    initial_zoom_estimates: List of N zoom estimates.
    base_focal_length: Tuple (fx, fy).
    principal_point: Tuple (cx, cy).
    """
    N = len(images)
    # Assume first N_out images are zoomed-out, rest are zoomed-in
    # In practice, N_out is determined by sorting zoom estimates or user input
    N_out = sum(1 for z in initial_zoom_estimates if z <= 1.1) # Heuristic
    N_in = N - N_out
    
    # Phase A: Global Pose Bootstrap
    # Initialize parameters
    theta = random_init_nerf()
    poses = []
    for i in range(N):
        if i < N_out:
            # Zoomed-out: init zoom to 1, random pose
            R = random_rotation()
            t = random_translation()
            xi = 1.0
            poses.append({'R': R, 't': t, 'xi': xi})
        else:
            # Zoomed-in: not used in Phase A loss, but need placeholder
            poses.append({'R': None, 't': None, 'xi': initial_zoom_estimates[i]})
            
    # Minimize L_coarse
    for iteration in range(phase_a_iters):
        loss = 0.0
        grad_theta = zeros_like(theta)
        grad_poses = [zeros_like(p) for p in poses]
        
        for i in range(N_out):
            # Render image i
            cam_params = {
                'R': poses[i]['R'],
                't': poses[i]['t'],
                'f': base_focal_length,
                'c': principal_point,
                'xi': poses[i]['xi']
            }
            rendered = render_nerf(theta, cam_params)
            error = rendered - images[i]
            loss += sum(error**2)
            
            # Compute gradients (backprop through renderer)
            d_loss_d_theta = grad_nerf(theta, cam_params, error)
            d_loss_d_pose = grad_pose(theta, cam_params, error)
            
            grad_theta += d_loss_d_theta
            grad_poses[i]['R'] += d_loss_d_pose['R']
            grad_poses[i]['t'] += d_loss_d_pose['t']
            grad_poses[i]['xi'] += d_loss_d_pose['xi']
            
        # Update parameters
        theta -= step_size * grad_theta
        for i in range(N_out):
            poses[i]['R'] -= step_size * grad_poses[i]['R']
            poses[i]['t'] -= step_size * grad_poses[i]['t']
            poses[i]['xi'] -= step_size * grad_poses[i]['xi']
            
    theta_A = theta
    poses_A = poses[:N_out]
    
    # Phase B: Cross-Zoom Pose Registration
    # Freeze theta
    theta_B = theta_A
    
    # Initialize poses for zoomed-in images using pose priming
    for j in range(N_out, N):
        # Find best matching zoomed-out image g*
        best_g = -1
        min_mse = infinity
        for g in range(N_out):
            # Create surrogate view
            # Extract central 1/xi_j crop from image[g]
            # Resize to native resolution of image[j]
            surrogate = create_surrogate(images[g], poses[g]['xi'], images[j].shape, poses[j]['xi'])
            mse = sum((surrogate - images[j])**2)
            if mse < min_mse:
                min_mse = mse
                best_g = g
                
        # Priming: Copy pose from best_g
        poses[j]['R'] = poses[best_g]['R']
        poses[j]['t'] = poses[best_g]['t']
        poses[j]['xi'] = initial_zoom_estimates[j] # Or optimize xi_j too? Source says optimize xi_j
        
    # Optimize poses for zoomed-in images
    for iteration in range(phase_b_iters):
        loss = 0.0
        grad_poses_in = [zeros_like(p) for p in poses[N_out:]]
        
        for j in range(N_out, N):
            cam_params = {
                'R': poses[j]['R'],
                't': poses[j]['t'],
                'f': base_focal_length,
                'c': principal_point,
                'xi': poses[j]['xi']
            }
            rendered = render_nerf(theta_B, cam_params)
            error = rendered - images[j]
            loss += sum(error**2)
            
            # Only optimize pose, theta is frozen
            d_loss_d_pose = grad_pose(theta_B, cam_params, error)
            
            grad_poses_in[j-N_out]['R'] += d_loss_d_pose['R']
            grad_poses_in[j-N_out]['t'] += d_loss_d_pose['t']
            grad_poses_in[j-N_out]['xi'] += d_loss_d_pose['xi']
            
        # Update poses for zoomed-in images
        for idx, j in enumerate(range(N_out, N)):
            poses[j]['R'] -= step_size * grad_poses_in[idx]['R']
            poses[j]['t'] -= step_size * grad_poses_in[idx]['t']
            poses[j]['xi'] -= step_size * grad_poses_in[idx]['xi']
            
    poses_B = poses # Full set of poses after Phase B
    
    # Phase C: Joint Refinement
    theta_C = theta_A
    poses_C = poses_B
    
    for iteration in range(phase_c_iters):
        loss = 0.0
        grad_theta = zeros_like(theta_C)
        grad_poses = [zeros_like(p) for p in poses_C]
        
        for k in range(N):
            cam_params = {
                'R': poses_C[k]['R'],
                't': poses_C[k]['t'],
                'f': base_focal_length,
                'c': principal_point,
                'xi': poses_C[k]['xi']
            }
            rendered = render_nerf(theta_C, cam_params)
            error = rendered - images[k]
            loss += sum(error**2)
            
            d_loss_d_theta = grad_nerf(theta_C, cam_params, error)
            d_loss_d_pose = grad_pose(theta_C, cam_params, error)
            
            grad_theta += d_loss_d_theta
            grad_poses[k]['R'] += d_loss_d_pose['R']
            grad_poses[k]['t'] += d_loss_d_pose['t']
            grad_poses[k]['xi'] += d_loss_d_pose['xi']
            
        # Update all parameters
        theta_C -= step_size * grad_theta
        for k in range(N):
            poses_C[k]['R'] -= step_size * grad_poses[k]['R']
            poses_C[k]['t'] -= step_size * grad_poses[k]['t']
            poses_C[k]['xi'] -= step_size * grad_poses[k]['xi']
            
    return theta_C, poses_C
```

### Edge Cases and Pitfalls
*   **Instability:** Simultaneous optimization of NeRF parameters and camera poses across all zoom levels is unstable. The three-phase schedule is required [S1].
*   **Unidentifiable Coupling:** Standard NeRF formulations assume fixed or small zoom-level perturbations. When magnification changes significantly, translation, scale, and focal length become coupled in an unidentifiable way, causing pose-free NeRFs to fail [S1].
*   **Pose Priming:** Random initialization for zoomed-in views leads to $O(\sigma_{rand}^2)$ initial error. Pose priming reduces this to $O(\delta^2)$ where $\delta$ is the crop misregistration error, ensuring convergence within the local strong convexity neighborhood [S1].

## Operating characteristics
*   **Time Complexity:** The method requires three sequential optimization phases. Phase B involves matching each zoomed-in image to a zoomed-out image, adding an $O(N_{in} \cdot N_{out})$ matching cost per iteration if done naively, though typically done once for initialization. The core optimization cost is dominated by the volume rendering integration in each phase.
*   **Memory Complexity:** Standard NeRF memory usage for MLP weights plus camera parameters. No additional large data structures like grids or pyramids are required [S1].
*   **Performance:** MZEN outperforms ablations that skip Phase B or train only on zoomed-in images [S1]. It achieves stable reconstruction where standard pose-free NeRFs fail [S1].
*   **Strengths:** Does not require high-resolution images or external pose initialization (SfM). Handles large zoom variations explicitly.
*   **Limitations:** Requires a clear distinction between zoomed-out and zoomed-in views. Relies on the assumption that zoomed-out views provide a stable global scaffold.

## Background
MZEN was introduced to address the limitations of Structure-from-Motion (SfM) and standard NeRF in industrial imaging systems (optical microscopes, digital inspection cameras, SEMs) where exact camera calibration is unavailable and the specimen stage moves only microns [S1]. Classical SfM fails in these conditions, and standard NeRF assumes fixed or small zoom-level perturbations [S1]. The method was published in *Machine Learning* (2026) [S1].

## Sources
[S1] MZEN: Multi-zoom Enhanced NeRF for 3-D Reconstruction with Unknown Camera Poses, https://link.springer.com/article/10.1007/s10994-026-07114-x, fetched 2026-10-26
