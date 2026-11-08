# L2G-NeRF

Also called: Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields

## Overview
L2G-NeRF is a bundle-adjusting method for Neural Radiance Fields that jointly optimizes neural 3D representations and camera poses by combining a local photometric alignment objective with a global geometric alignment objective [S1]. It addresses the sensitivity of joint optimization to poor initialization by using a two-stage process: first, a non-parametric pixel-wise flexible alignment minimizes photometric error; second, a parametric frame-wise global alignment constrains these local transformations to obey rigid geometric constraints [S1]. Unlike pure parametric methods like BARF, which optimize global poses directly and diverge with large initial errors, L2G-NeRF tolerates larger noise perturbations by decoupling the local photometric fit from the global geometric consistency [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- A set of $M$ images $\{I_i\}_{i=1}^M$ [S1].
- Query pixel coordinates $\{x_j\}_{j=1}^N$ for each image [S1].
- Initial camera poses $T_i$, initialized to identity $I$ with random perturbations [S1].

**Outputs:**
- Neural field parameters $\Theta$ [S1].
- Optimized camera poses $\{T_i\}_{i=1}^M$ [S1].
- Warp field parameters $\Phi$ and frame embeddings $\{\ell_i\}_{i=1}^M$ [S1].

### Data Structures
1.  **Neural Field MLP ($f_R$):**
    -   **Input:** 3D coordinate $z \in \mathbb{R}^3$ (for 3D NeRF) or 2D coordinate $x \in \mathbb{R}^2$ (for 2D images) [S1].
    -   **Output:** 4D vector $[c; \sigma] \in \mathbb{R}^4$ (RGB color $c$ and volume density $\sigma$) for 3D NeRF, or 3D vector for 2D images [S1].
    -   **Parameters:** $\Theta$ [S1].
2.  **Warp Neural Field MLP ($f_W$):**
    -   **Input:** 2D coordinate $x \in \mathbb{R}^2$ and frame-dependent embedding $\ell_i \in \mathbb{R}^d$ [S1].
    -   **Output:** Lie algebra vector (se(2) or sl(3)) [S1].
    -   **Parameters:** $\Phi$ [S1].
3.  **Camera Pose ($T_i$):**
    -   Parametrized as $T = [R|t] \in SE(3)$, where $R \in SO(3)$ and $t \in \mathbb{R}^3$ [S1].
    -   Rotation $R$ is generated via exponential map $R = \exp(r)$ from Lie algebra $\mathfrak{so}(3)$ [S1].

### Numbered Steps

1.  **Initialization:**
    -   Initialize all camera poses $T_i = I$ [S1].
    -   Perturb initial poses using right multiplication to alter viewing directions and distances [S1].
    -   Additive rotation noise $r \sim \mathcal{N}(0, n_r I)$ and translation noise $t \sim \mathcal{N}(0, n_t I)$ are applied [S1].
    -   Base noise multipliers $n_r$ and $n_t$ are scene-dependent (e.g., $n_r \in [0.01, 0.09]$, $n_t \in [0.3, 0.7]$) [S1].

2.  **Stage 1: Local Alignment (Non-Parametric):**
    -   Model pixel-wise local transformations $T_{ji}$ using a warp neural field [S1]:
        $$ T_{ji} = W(x_j; \ell_i, \Phi) = \exp(f_W(x_j; \ell_i, \Phi)) $$
        where $\exp(\cdot)$ maps the Lie algebra output to the Lie group SE(2) or SL(3) [S1].
    -   Optimize $\Phi$, $\{\ell_i\}$, and $\Theta$ by minimizing the local photometric objective:
        $$ \mathcal{L}_{local} = \sum_{i=1}^M \sum_{j=1}^N \| R(T_{ji} x_j; \Theta) - I_i(x_j) \|^2 $$
    -   For 3D NeRF, the rendering $R(T_{ji} x_j; \Theta)$ is computed via volume rendering [S1]:
        $$ R(x) = \int_{z_{near}}^{z_{far}} T(x, z) \sigma(zx) c(zx) dz $$
        Discretized with $K$ points sampled along the ray at depths $\{z_1, ..., z_K\}$:
        $$ R(x) = g(y_1, ..., y_K), \quad y_k = f_R(z_k T_{ji} x_j; \Theta) $$
        where $g$ is the differentiable compositing function [S1].

3.  **Stage 2: Global Alignment (Parametric):**
    -   Solve for the frame-wise global transformation $T^*_i$ for each image $i$ by minimizing the alignment error between local and global transformations (Orthogonal Procrustes problem) [S1]:
        $$ T^*_i = \arg \min_{T_i} \sum_{j=1}^N \| T_{ji} x_j - T_i x_j \|^2 $$
    -   The optimal rigid transformation $T=[R|t]$ is computed in closed form [S1]:
        -   Compute centroids $\bar{x} = \frac{1}{N}\sum x_j$ and $\overline{Tx} = \frac{1}{N}\sum T_{ji} x_j$ [S1].
        -   Compute covariance matrix $H = \sum_{j=1}^N (x_j - \bar{x})(T_{ji} x_j - \overline{Tx})^T$ [S1].
        -   Perform SVD: $H = U S V^T$ [S1].
        -   Calculate $R = V U^T$ and $t = -R\bar{x} + \overline{Tx}$ [S1].
    -   Apply $T^*_i$ as a soft constraint to the local alignment by adding the global objective to the loss [S1]:
        $$ \mathcal{L}_{global} = \lambda \sum_{i=1}^M \sum_{j=1}^N \| T_{ji} x_j - T^*_i x_j \|^2 $$
    -   Jointly optimize $\Theta$, $\Phi$, $\{\ell_i\}$, and $\{T_i\}$ by minimizing the combined objective:
        $$ \min_{\{T_{ji}\}, \{T_i\}, \Theta, \Phi, \{\ell_i\}} \mathcal{L}_{local} + \mathcal{L}_{global} $$

4.  **Convergence Schedule:**
    -   Linearly increase noise multipliers $n_r$ and $n_t$ by a common factor $\{n_c\}_{c=2}^9$ starting from base values $n_r=0.01, n_t=0.1$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def exponential_map_so3(r):
    """
    Maps Lie algebra vector r (3,) to rotation matrix R (3,3).
    Uses Rodrigues' formula.
    """
    theta = np.linalg.norm(r)
    if theta < 1e-8:
        return np.eye(3)
    k = r / theta
    K = np.array([[0, -k[2], k[1]],
                  [k[2], 0, -k[0]],
                  [-k[1], k[0], 0]])
    return np.eye(3) + np.sin(theta) * K + (1 - np.cos(theta)) * K @ K

def volume_render(f_R, points_3d, K):
    """
    Renders color from neural field f_R at given 3D points.
    f_R: function mapping (3,) -> (4,) [c, sigma]
    points_3d: (K, 3) array of points along ray
    K: number of samples
    """
    # Evaluate MLP at all points
    # y_k = [c_k, sigma_k]
    outputs = np.array([f_R(p) for p in points_3d]) # Shape (K, 4)
    c = outputs[:, :3]
    sigma = outputs[:, 3]
    
    # Compute weights
    # alpha_k = 1 - exp(-sigma_k * delta)
    # Assuming uniform spacing or precomputed deltas
    # Standard NeRF compositing
    alpha = 1.0 - np.exp(-sigma)
    T_prev = np.ones(K + 1)
    T_prev[1:] = np.cumprod(1.0 - alpha)
    
    weights = alpha * T_prev[:-1]
    color = np.sum(weights[:, np.newaxis] * c, axis=0)
    return color

def solve_procrustes(points_source, points_target):
    """
    Solves Orthogonal Procrustes problem: min ||R x + t - y||^2
    points_source: (N, 3) local transformed points T_ji x_j
    points_target: (N, 3) query points x_j (homogeneous or 3D depending on context)
    Returns R (3,3), t (3,)
    """
    # Note: In 2D image alignment, points are 2D. In 3D NeRF, 
    # the paper describes solving for T_i in SE(3) based on correspondences.
    # The formula provided in S1 is for the classic Procrustes problem.
    
    N = points_source.shape[0]
    
    # Centroids
    x_bar = np.mean(points_target, axis=0)
    y_bar = np.mean(points_source, axis=0)
    
    # Centered points
    X_c = points_target - x_bar
    Y_c = points_source - y_bar
    
    # Covariance matrix H
    H = X_c.T @ Y_c
    
    # SVD
    U, S, Vt = np.linalg.svd(H)
    V = Vt.T
    
    # Rotation
    R = V @ U.T
    
    # Check for reflection
    if np.linalg.det(R) < 0:
        V[:, -1] *= -1
        R = V @ U.T
        
    # Translation
    t = y_bar - R @ x_bar
    
    return R, t

def l2g_nerf_train(images, initial_poses, f_R, f_W, K, lambda_global, n_r_base, n_t_base):
    """
    Main training loop for L2G-NeRF.
    """
    M = len(images)
    N = images[0].shape[0] # Number of pixels
    
    # Initialize parameters
    Theta = init_params_f_R()
    Phi = init_params_f_W()
    embeddings = [init_embedding() for _ in range(M)]
    
    # Initialize poses with perturbation
    poses = []
    for i in range(M):
        R_init = np.eye(3)
        t_init = np.zeros(3)
        # Add noise
        r_noise = np.random.normal(0, n_r_base, 3)
        t_noise = np.random.normal(0, n_t_base, 3)
        R_init = exponential_map_so3(r_noise) @ R_init # Right multiplication
        t_init = t_init + t_noise
        poses.append((R_init, t_init))
        
    # Optimization loop
    for epoch in range(num_epochs):
        # Update noise multipliers (convergence schedule)
        n_r = n_r_base * (1 + epoch * factor)
        n_t = n_t_base * (1 + epoch * factor)
        
        for i in range(M):
            I_i = images[i]
            R_i, t_i = poses[i]
            
            # 1. Local Alignment: Predict local transformations
            # For each pixel j, compute local transform T_ji
            local_transforms = []
            for j in range(N):
                x_j = get_pixel_coord(j) # 2D or 3D depending on mode
                # Warp field input: 2D coord + embedding
                # Output: Lie algebra vector
                lie_vec = f_W(x_j, embeddings[i], Phi)
                # Exponentiate to get transformation matrix
                # Note: Implementation details for exp(se2/sl3) depend on dimension
                T_ji = exp_lie(lie_vec) 
                local_transforms.append(T_ji)
            
            # 2. Global Alignment: Solve for global T*_i
            # Correspondences: source = T_ji * x_j, target = x_j (in camera space)
            # Or in world space? The paper says T*_i minimizes ||T_ji x_j - T_i x_j||
            # This implies T_ji and T_i map x_j to the same space.
            # Assuming x_j are homogeneous coordinates in camera view.
            
            points_source = []
            points_target = []
            for j in range(N):
                x_j = get_pixel_coord(j)
                # Transform x_j by local T_ji
                p_source = T_ji @ x_j
                # Target is x_j transformed by current global pose estimate?
                # The objective is ||T_ji x_j - T_i x_j||.
                # So target is T_i x_j. But T_i is being optimized.
                # The Procrustes solver finds the best T_i given fixed T_ji.
                # So we treat T_ji x_j as "source" and x_j as "target" 
                # to find the rigid transform that maps x_j to T_ji x_j?
                # No, T_i is the global pose. T_ji is the local pose.
                # We want T_i such that T_i x_j approx T_ji x_j.
                # So Source = T_ji x_j, Target = x_j? 
                # If T_i = I, then we map x_j to T_ji x_j.
                # So yes, Source = T_ji x_j, Target = x_j.
                points_source.append(p_source)
                points_target.append(x_j)
                
            points_source = np.array(points_source)
            points_target = np.array(points_target)
            
            R_star, t_star = solve_procrustes(points_source, points_target)
            T_star = (R_star, t_star)
            
            # 3. Compute Losses
            # Local Loss: Photometric error
            loss_local = 0
            for j in range(N):
                x_j = get_pixel_coord(j)
                # Apply local transform to ray origin/direction or 3D points
                # In 3D NeRF, T_ji transforms the 3D point z_k x_j
                points_3d_local = []
                for k in range(K):
                    z_k = get_depth(k)
                    p_3d = z_k * (T_ji @ x_j) # Homogeneous to 3D?
                    # Actually, T_ji is SE(3). x_j is homogeneous 3D.
                    # p_3d = T_ji @ x_j gives 3D point in world?
                    # NeRF expects points in world space.
                    points_3d_local.append(p_3d)
                
                color_pred = volume_render(f_R, np.array(points_3d_local), K)
                loss_local += np.sum((color_pred - I_i[j])**2)
            
            # Global Loss: Geometric constraint
            loss_global = 0
            for j in range(N):
                x_j = get_pixel_coord(j)
                p_local = local_transforms[j] @ x_j
                p_global = T_star @ x_j
                loss_global += np.sum((p_local - p_global)**2)
                
            total_loss = loss_local + lambda_global * loss_global
            
            # Backpropagate and update Theta, Phi, embeddings, poses
            update_parameters(Theta, Phi, embeddings, poses, total_loss)
            
    return Theta, poses
```

### Parameters
-   $\lambda$: Weighting factor for the global constraint term. Set to $1 \times 10^2$ for synthetic/object-centric scenes; floated between $1 \times 10^2$ and $1 \times 10^5$ for forward-facing LLFF scenes [S1].
-   $n_r$: Base rotation noise multiplier. Scene-dependent values range from 0.01 to 0.09 [S1].
-   $n_t$: Base translation noise multiplier. Scene-dependent values range from 0.3 to 0.7 [S1].
-   $K$: Number of points sampled along a ray for discretizing the volume rendering integral [S1].
-   $\Theta$: Parameters of the neural field $f_R$ [S1].
-   $\Phi$: Parameters of the warp neural field $f_W$ [S1].
-   $\ell_i$: Frame-dependent embedding for frame $i$ [S1].

### Edge Cases and Pitfalls
-   **Poor Initialization:** Parametric methods (directly optimizing global transformations) fall into suboptimal solutions if poorly initialized [S1]. L2G-NeRF mitigates this but still fails to recover camera poses from identity transformation for inward-facing 360° scenes with large rotation displacements [S1].
-   **Large $\lambda$:** A large $\lambda$ ($1 \times 10^5$) improves geometric accuracy but reduces robustness, causing divergence on difficult scenes like "Flower" [S1].
-   **Non-Parametric Limitations:** Non-parametric methods (optimizing pixel agreement) have trouble dealing with large displacements as alignments may not obey geometric constraints [S1].
-   **Divergence:** Removing the global alignment objective ($\lambda=0$) causes divergence to suboptimal poses and ghosting artifacts [S1].

## Operating characteristics
-   **Time Complexity:** L2G-NeRF is more efficient than BARF because it computes gradients with respect to local pose rather than global pose [S1].
-   **Measured Performance:** On an NVIDIA GeForce RTX 2080 Ti, L2G-NeRF takes ~4.5 hours for synthetic objects and ~8 hours for real-world scenes, compared to ~8 and ~10.5 hours for BARF [S1].
-   **Robustness:** L2G-NeRF tolerates larger noise perturbations before divergence ($n_c=8$ vs $n_c=4$ for BARF) [S1].
-   **Strengths:** High-fidelity reconstruction, resolves large camera pose misalignment, robust to poor initialization [S1].
-   **Limitations:** Fails on inward-facing 360° scenes with large rotation displacements from identity initialization [S1].

## Background
L2G-NeRF was proposed in the paper "Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields" [S1]. It builds on the problem of bundle-adjusting neural fields, which is a type of classic photometric bundle adjustment [S1]. The method combines parametric and non-parametric registration techniques to overcome the limitations of each when used alone [S1].

## Sources
[S1] Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00799, fetched 2026-10-29
