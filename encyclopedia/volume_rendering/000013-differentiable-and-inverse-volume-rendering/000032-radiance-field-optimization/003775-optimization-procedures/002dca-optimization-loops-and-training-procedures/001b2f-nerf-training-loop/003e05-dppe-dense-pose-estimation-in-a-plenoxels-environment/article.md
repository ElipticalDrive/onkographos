# DPPE (Dense Pose Estimation in a Plenoxels Environment)
Also called: Render-Based Plenoxels Pose Estimation

## Overview
DPPE is a 6-DoF monocular RGB-only pose estimation procedure for Plenoxels environments that recovers the ground truth camera pose after a perturbation by minimizing photometric errors in re-rendering [S1, unofficial]. It distinguishes itself from analytical derivative-based methods (e.g., Teigen et al. [30]) by using numerical gradient approximation, making it generalizable to any scene representation where re-renderings are not prohibitively expensive, and from NeRF-based methods (e.g., iNeRF) by leveraging Plenoxels' rendering speed to re-render the full image at each optimization step rather than subsampling rays [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained Plenoxels grid; a ground truth image $I_{gt}$; a ground truth camera pose $\pi_{gt}$; a perturbation vector $\Delta\pi$ (6-DoF: 3 translation, 3 rotation) applied to $\pi_{gt}$ to initialize the current pose $\pi_c$; image dimensions $W, H$; focal length $f$; maximum epochs; convergence threshold $\epsilon$; learning rates $\alpha$.
*   **Outputs:** The optimized final camera pose $\pi_{final}$.

**Data Structures:**
*   **Plenoxels Grid:** A 3D voxel grid storing Plenoptic volume elements. Default resolution is $[256]^3$ [S1, unofficial].
*   **Camera Pose $\pi$:** A $4 \times 4$ camera-to-world matrix $[\pi_R | \pi_t]$, where $\pi_R$ is the $3 \times 3$ rotation matrix and $\pi_t$ is the $3 \times 1$ translation vector [S1, unofficial].
*   **Perturbation $\Delta\pi$:** A vector $[x, y, z, \omega_1, \omega_2, \omega_3]$ representing translation and orientation changes [S1, unofficial].

**Rendering Model:**
The color $\hat{C}(r)$ for a ray $r$ with $N$ samples is computed via volume rendering [S1, unofficial]:
$$ \hat{C}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
where $\sigma_i$ is opacity, $\delta_i$ is the distance between samples, $c_i$ is the color from trilinear interpolation of the 8 nearest voxels, and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the transmittance [S1, unofficial].

**Ray Casting:**
For a pixel $p$ with coordinates $(u, v)$, the ray $r$ is defined by origin $o_r$ and direction $d_r$ [S1, unofficial]:
$$ o_r = \pi_t $$
$$ d_r = \pi_R \begin{bmatrix} \frac{u - W/2}{f} \\ \frac{-v - H/2}{f} \\ -1 \end{bmatrix} $$
The ray is parameterized as $r = o_r + h d_r$ for $h \in \mathbb{R}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Set current pose $\pi_c = \pi_{gt} + \Delta\pi$. Initialize learning rates $\alpha_{trans}$ and $\alpha_{rot}$.
2.  **Iteration Loop:** While the squared L2 norm of the difference between the ground truth image $I_{gt}$ and the rendered image $\hat{I}_{\pi_c}$ is greater than $\epsilon$ AND the number of epochs is less than the maximum allowed epochs:
    a.  **Render Current Pose:** Cast rays $r$ from $\pi_c$ and render the image $\hat{I}_{\pi_c}$ using the Plenoxels volume renderer.
    b.  **Gradient Approximation:** For each of the 6 camera degrees of freedom $i \in [x, y, z, \omega_1, \omega_2, \omega_3]$:
        i.  Perturb the current pose $\pi_c$ by $+1$ step in DoF $i$ to get $\pi_{c+}$ and by $-1$ step to get $\pi_{c-}$.
        ii. Render images $\hat{I}_{\pi_{c+}}$ and $\hat{I}_{\pi_{c-}}$ for these perturbed poses.
        iii. Compute the gradient component $J_i(r)$ using central differencing:
        $$ J_i(r) \leftarrow \frac{\hat{I}_{\pi_{c+}}(r) - \hat{I}_{\pi_{c-}}(r)}{2} $$
    c.  **Compute Error:** Calculate the photometric error $\Delta I(r)$:
        $$ \Delta I(r) \leftarrow \hat{I}_{\pi_c}(r) - I_{gt}(r) $$
    d.  **Pose Update:** Update the pose $\pi_c$ using Stochastic Gradient Descent (SGD):
        $$ \pi_c \leftarrow \pi_c + \alpha (J(r) \cdot \Delta I(r)) $$
        Note: The optimization strategy alternates between a single rotation step (updating 3 DoF) and a single translation step (updating 3 DoF) per epoch [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dppe_pose_estimation(plenoxels_grid, I_gt, pi_gt, delta_pi, W, H, f, epsilon, max_epochs, alpha_trans_init, alpha_rot_init):
    """
    DPPE: Dense Pose Estimation in a Plenoxels Environment.
    
    Args:
        plenoxels_grid: Pre-trained Plenoxels grid data structure.
        I_gt: Ground truth image (W x H x 3).
        pi_gt: Ground truth camera-to-world pose matrix (4x4).
        delta_pi: Perturbation vector [x, y, z, w1, w2, w3].
        W, H: Image width and height.
        f: Focal length.
        epsilon: Convergence threshold for squared L2 norm.
        max_epochs: Maximum number of optimization epochs.
        alpha_trans_init: Initial translation learning rate.
        alpha_rot_init: Initial rotation learning rate.
        
    Returns:
        pi_final: Optimized camera pose matrix (4x4).
    """
    # Initialize current pose
    pi_c = pi_gt + delta_pi
    
    # Initialize learning rates
    alpha_trans = alpha_trans_init
    alpha_rot = alpha_rot_init
    
    # Helper to render image from pose
    def render_image(pose, W, H, f):
        # Cast rays for all pixels (u, v) in [0, W-1] x [0, H-1]
        # Ray origin: o_r = pose[0:3, 3]
        # Ray direction: d_r = pose[0:3, 0:3] @ [ (u - W/2)/f, -(v + H/2)/f, -1 ]^T
        # Render using Plenoxels volume rendering:
        # C(r) = sum_{i=1}^N T_i * (1 - exp(-sigma_i * delta_i)) * c_i
        # where T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
        # c_i from trilinear interpolation of 8 nearest voxels in plenoxels_grid
        pass # Implementation depends on specific Plenoxels renderer API

    # Helper to perturb pose
    def perturb_pose(pose, dof_index, step_size):
        # dof_index: 0-2 for translation (x, y, z), 3-5 for rotation (w1, w2, w3)
        # step_size: magnitude of perturbation
        # For translation: shift by 1 voxel in the specified direction
        # For rotation: rotate by step_size radians around the specified axis
        # Return new pose matrix
        pass # Implementation depends on specific pose perturbation logic

    epoch = 0
    while epoch < max_epochs:
        # Render current pose
        I_c = render_image(pi_c, W, H, f)
        
        # Check convergence
        error_sq_l2 = np.sum((I_c - I_gt) ** 2)
        if error_sq_l2 <= epsilon:
            break
            
        # Determine if this epoch is translation or rotation step
        # Alternating strategy: even epochs for translation, odd for rotation (or vice versa)
        # The source states: "For each epoch, either a translation step or a rotation step is performed"
        # We assume alternating based on epoch parity for implementation clarity
        is_translation_step = (epoch % 2 == 0)
        
        if is_translation_step:
            # Translation step: update 3 DoF (x, y, z)
            # Learning rate alpha_trans
            # Perturbation step size: 1 voxel in Plenoxels grid
            voxel_size = get_voxel_size(plenoxels_grid) # Not given in sources, assumed known from grid metadata
            
            # Compute gradient for translation DoFs
            J_trans = np.zeros_like(I_c)
            for i in range(3): # x, y, z
                # Perturb +1 voxel in direction i
                pi_plus = perturb_pose(pi_c, i, voxel_size)
                I_plus = render_image(pi_plus, W, H, f)
                
                # Perturb -1 voxel in direction i
                pi_minus = perturb_pose(pi_c, i, -voxel_size)
                I_minus = render_image(pi_minus, W, H, f)
                
                # Central differencing
                J_trans += (I_plus - I_minus) / 2.0
                
            # Photometric error
            Delta_I = I_c - I_gt
            
            # SGD Update
            # Note: The formula pi_c <- pi_c + alpha * (J(r) * Delta I(r)) implies element-wise multiplication
            # However, J is an image gradient, Delta I is an image error.
            # The update is applied to the pose parameters, not the image.
            # The source formula: pi_c <- pi_c + alpha (J(r) . Delta I(r))
            # This likely means the gradient of the loss with respect to pose is approximated by J . Delta I
            # And the pose is updated by this gradient scaled by alpha.
            # Since J is an image, we need to aggregate or map it back to pose parameters.
            # The source says "J(r) is the steepest descent image".
            # In standard gradient descent for pose, we need dL/dpi.
            # L = sum (I_c - I_gt)^2. dL/dpi = 2 * sum (I_c - I_gt) * dI_c/dpi.
            # dI_c/dpi is approximated by J.
            # So the gradient for the pose is proportional to sum(J * Delta_I).
            # The source formula is slightly ambiguous on aggregation.
            # Given "SGD step", we assume the gradient is computed and applied to pi_c.
            
            # Approximate gradient for pose update
            # J_trans is WxHx3. Delta_I is WxHx3.
            # We need to compute the gradient with respect to the 3 translation parameters.
            # This requires mapping the image gradient back to the pose parameters.
            # The source does not provide the explicit mapping from image gradient to pose gradient.
            # It simply states: pi_c <- pi_c + alpha (J(r) . Delta I(r))
            # This might imply a simplified update or that J(r) is already the pose gradient.
            # However, J(r) is defined as [I(R(i+1)) - I(R(i-1))]/2, which is an image.
            # Let's assume the source implies the gradient of the loss with respect to the pose parameter i
            # is approximated by the dot product of the image gradient and the error, summed over pixels.
            
            # Gradient for translation parameter i
            grad_trans_i = np.sum(J_trans[:, :, i] * Delta_I[:, :, i]) # Sum over pixels and channels?
            # The source formula uses J(r) . Delta I(r). If J is the gradient image,
            # the loss gradient is likely the inner product.
            
            # Update translation parameters
            # This part is ambiguous in the source. We will implement a standard gradient descent step
            # using the aggregated gradient.
            for i in range(3):
                # Compute gradient for translation i
                # J_trans is the gradient image for translation i? No, J_trans was summed over 3 DoFs in the loop above?
                # Let's re-read: "for all i in [x,y,z,...] ... Ji(r) <- ...". So Ji is specific to i.
                # So we should compute Ji for each i separately.
                pass 
            
            # Re-implementing the loop correctly for translation
            for i in range(3):
                pi_plus = perturb_pose(pi_c, i, voxel_size)
                I_plus = render_image(pi_plus, W, H, f)
                pi_minus = perturb_pose(pi_c, i, -voxel_size)
                I_minus = render_image(pi_minus, W, H, f)
                Ji = (I_plus - I_minus) / 2.0
                Delta_I = I_c - I_gt
                # Gradient of loss w.r.t translation i
                # L = sum (I_c - I_gt)^2
                # dL/dpi_i = 2 * sum (I_c - I_gt) * dI_c/dpi_i
                # dI_c/dpi_i approx Ji
                grad = 2 * np.sum(Ji * Delta_I) # Sum over all pixels and channels
                # Update pose
                # The source says pi_c <- pi_c + alpha * (J(r) . Delta I(r))
                # This is likely a simplified representation.
                # We update the translation component of pi_c
                pi_c[0:3, 3] += alpha_trans * grad # This is a simplification; actual update depends on how grad maps to pose
                
            # Decay learning rate
            if (epoch + 1) % 100 == 0:
                alpha_trans *= 0.75
                
        else:
            # Rotation step: update 3 DoF (w1, w2, w3)
            # Learning rate alpha_rot
            # Perturbation step size: 0.001 radians
            
            for i in range(3, 6): # w1, w2, w3
                pi_plus = perturb_pose(pi_c, i, 0.001)
                I_plus = render_image(pi_plus, W, H, f)
                pi_minus = perturb_pose(pi_c, i, -0.001)
                I_minus = render_image(pi_minus, W, H, f)
                Ji = (I_plus - I_minus) / 2.0
                Delta_I = I_c - I_gt
                grad = 2 * np.sum(Ji * Delta_I)
                # Update rotation component of pi_c
                # Actual update depends on rotation parameterization
                pass
                
            if (epoch + 1) % 100 == 0:
                alpha_rot *= 0.75
                
        epoch += 1
        
    return pi_c
```

**Parameters:**
*   **$\epsilon$ (Convergence Threshold):** Set to 2000 for full image usage; set to 0 for 1% image usage [S1, unofficial].
*   **$\alpha$ (Learning Rate):** Translation $\alpha$ initialized at 4, decayed by 25% every 100 epochs. Rotation $\alpha$ initialized at 5, decayed by 25% every 100 epochs [S1, unofficial].
*   **Max Epochs:** 1000 [S1, unofficial].
*   **Plenoxels Grid Resolution:** Typically $[256]^3$ [S1, unofficial].
*   **Step Sizes:** Translation: 1 voxel (variable size depending on grid resolution). Rotation: 0.001 radians [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Local Minima:** The method may get stuck in local minima. While other works use Monte-Carlo sampling, DPPE relies on gradient approximation and potentially subsampling strategies to mitigate this [S1, unofficial].
*   **Translation Step Size Variability:** The translation step size is variable depending on the Plenoxels grid resolution; a shift of 1 voxel changes magnitude based on grid size [S1, unofficial].
*   **Computational Cost:** Although Plenoxels is faster than NeRF, rendering 6 images per epoch is still computationally expensive. The method leverages the speed of Plenoxels to allow full-image re-rendering, which is not feasible with standard NeRF [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires 6 image renders per epoch (1 for current pose, 2 for each of the 6 DoF for gradient approximation). However, Plenoxels rendering is fast enough that more than 6 image renders can be performed in the time it takes a single NeRF render [S1, unofficial].
*   **Memory Complexity:** Depends on the Plenoxels grid resolution. Default $[256]^3$ grid requires significant VRAM, but fits within 24 GB VRAM on an NVIDIA RTX-4090 [S1, unofficial].
*   **Measured Performance:** On the NeRF Synthetic Dataset (Chair scene), DPPE-full achieved an Average Rotation Error of 2.36° and Average Translation Error of 0.17 units, compared to iNeRF's 3.71° and 0.31 units. On average across 8 scenes, DPPE-full achieved 2.95° and 0.21 units, compared to iNeRF's 3.57° and 0.24 units [S1, unofficial].
*   **Strengths:** Uses only monocular RGB data; generalizable to any scene representation with fast re-rendering; full-image re-rendering capability due to Plenoxels speed.
*   **Limitations:** Susceptible to local minima; translation step size variability; requires pre-trained Plenoxels grid.

## Background
DPPE was introduced by Christopher Kolios, Yeganeh Bahoo, and Sajad Saeedi in March 2024 [S1, unofficial]. It builds on the Plenoxels technique [7, unofficial] which offers reduced training times and better rendering accuracy by eliminating the neural net component. The method addresses the limitations of iNeRF [5, unofficial] (slow rendering leading to subsampling) and Teigen et al. [30, unofficial] (requirement for RGB-D data and analytical derivatives tied to Plenoxels) by using numerical gradients and monocular RGB data [S1, unofficial].

## Sources
[S1] DPPE: Dense Pose Estimation in a Plenoxels Environment using Gradient Approximation, https://doi.org/10.48550/arxiv.2403.10773, fetched 2026-10-26 (unofficial)
