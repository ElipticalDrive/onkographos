# Differential Rendering for CT Reconstruction

Also called: Differentiable Rendering for CT Reconstruction, Differentiable Volume Rendering Reconstruction, Gradient-based Volumetric Reconstruction, Inverse Rendering for CT, Iterative Differential CT

## Overview
Differential Rendering for CT Reconstruction is an inverse rendering technique that reconstructs the volumetric absorption coefficients of an object from a series of 2D X-ray projection images by optimizing scene parameters via gradient descent [S1, unofficial]. It formulates Computed Tomography (CT) as an iterative optimization problem where the Mean Squared Error (MSE) between differentially rendered images and reference projections is minimized [S1, unofficial]. Unlike traditional algebraic reconstruction techniques (ART/SIRT) or deep learning-based methods, this approach relies on fully auto-differentiable rendering engines (such as Mitsuba 2) to compute gradients directly from the rendering equation, allowing for end-to-end optimization without pre-trained models [S1, unofficial]. It sets itself apart from siblings like DiffVox by utilizing a specific multi-view iterative rotation strategy with a fixed 45° step size and ADAM optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference projection images (Ground Truth) acquired at various rotation angles; initial scene parameters $\pi$ including rotation angle and volume data $\sigma$; optimization hyperparameters ($N_{rot}, N_{iter}, LR, \beta_1, \beta_2, \epsilon$).
*   **Outputs:** Optimized 3D volume data (absorption coefficient matrix $\sigma$).

**Data Structures:**
*   **Voxel Grid:** A 3D grid representation of the object. The source uses a $16 \times 16 \times 16$ grid [S1, unofficial].
*   **Absorption Coefficient ($\sigma$):** A scalar value per voxel. Value 1 means fully absorbing; 0 means no absorption. Initial reference values are typically $10^{-4}$ for occupied voxels and 0 for empty space [S1, unofficial].
*   **Scene Parameters ($\pi$):** Contains the current rotation angle and the current state of the volume data $\sigma$ [S1, unofficial].
*   **Update Buffer ($\sigma_{update}$):** A non-absorbing matrix (volume data) used to update the scene parameters. Initialized to zero in the first run [S1, unofficial].

**Mathematical Formulation:**
The core principle is differentiating the rendering function with respect to the absorption coefficient matrix $\pi_0$ (representing $\sigma$) [S1, unofficial]:

$$
\frac{\partial}{\partial \pi_0} \iint f(x, y; \pi) \, dx \, dy = \frac{\partial I_i}{\partial \pi_0}
$$

Where:
*   $\pi_0$ is the absorption coefficient matrix of the volumetric object.
*   $f(x, y; \pi)$ is the rendering function mapping scene parameters to image coordinates.
*   $I_i$ is the pixel value in the rendered image.

**Algorithm Steps:**
1.  **Initialize:** Set $N_{rot}$ (number of 45° rotations) and $N_{iter}$ (iterations per rotation). Initialize the absorption coefficient parameter matrix $\pi_0$ with constant entries (e.g., 0 or $0.001$) [S1, unofficial].
2.  **Loop over Rotations ($i$ from 0 to $N_{rot}-1$):**
    a.  **Calculate Angle:** Compute rotation angle as $(i \times 45.0) \pmod{360}$ [S1, unofficial].
    b.  **Load Scene:** Create scene object using parameters $\pi$ and current rotation [S1, unofficial].
    c.  **Render Reference:** Forward render the scene to create the target image ($I_{ref}$) [S1, unofficial].
    d.  **Initialize Update Buffer:** If $\sigma_{update}$ is null, initialize it as a zero-filled array matching the length of the reference volume $\sigma$ data [S1, unofficial].
    e.  **Update Volume:** Apply $\sigma_{update}$ to the scene volume parameters [S1, unofficial].
    f.  **Loop over Iterations ($j$ from 0 to $N_{iter}-1$):**
        i.  **Differential Render:** Render the scene using the differentiable renderer to get the current image ($I_{curr}$) and compute gradients [S1, unofficial].
        ii. **Calculate Loss:** Compute MSE between $I_{curr}$ and $I_{ref}$:
        $$
        L = \frac{1}{N} \sum_{k=1}^{N} (I_{curr, k} - I_{ref, k})^2
        $$
        iii. **Backpropagate:** Compute gradients of the loss with respect to scene parameters ($\frac{\partial L}{\partial \pi}$) [S1, unofficial].
        iv. **Gradient Step:** Update parameters using the ADAM optimizer [S1, unofficial].
    g.  **Update Buffer:** After the iteration loop, update $\sigma_{update}$ with the optimized $\sigma$ data from the scene volume [S1, unofficial].
3.  **Convergence:** Repeat until $N_{rot}$ rotations are complete or convergence is reached (no further changes in scene parameters) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differential_ct_reconstruction(Nrot, Niter, initial_sigma, lr, beta1, beta2, epsilon):
    """
    Reconstructs volumetric data using differential rendering.
    
    Args:
        Nrot: Number of 45-degree rotations (e.g., 40 for 360 degrees).
        Niter: Number of optimization iterations per rotation (e.g., 40).
        initial_sigma: Initial absorption coefficient matrix (e.g., zeros or 0.001).
        lr: Learning rate for ADAM (0.002).
        beta1: ADAM beta1 (0.9).
        beta2: ADAM beta2 (0.99).
        epsilon: ADAM epsilon (1e-8).
        
    Returns:
        Optimized sigma volume.
    """
    # Initialize update buffer
    sigma_update = None
    
    # Scene parameters container
    # In a real implementation, this would hold rotation state and volume data
    scene_params = {
        'rotation': 0.0,
        'sigma': initial_sigma
    }
    
    for i in range(Nrot):
        # 1. Calculate rotation angle
        rotation_angle = (i * 45.0) % 360.0
        
        # 2. Setup scene with current rotation and parameters
        # load_scene handles camera, detector, and volume setup
        scene = load_scene(scene_params['sigma'], rotation_angle)
        
        # 3. Render reference image (Ground Truth)
        # This is typically done via forward rendering of the known/reference scene
        image_ref = render_forward(scene)
        
        # 4. Initialize sigma_update if null
        if sigma_update is None:
            sigma_update = np.zeros_like(scene_params['sigma'])
            
        # 5. Apply update buffer to scene volume
        scene.volume.sigma = sigma_update
        
        # 6. Optimization loop for this rotation
        for j in range(Niter):
            # Render differentially to get image and gradients
            # render_differential returns (image, gradients_of_loss_wrt_sigma)
            # Note: The loss is computed internally or explicitly as MSE
            image_curr, gradients = render_differential(scene, image_ref)
            
            # Compute MSE Loss explicitly for clarity (though ADAM step uses gradients)
            # loss = mean((image_curr - image_ref)^2)
            
            # Update parameters using ADAM
            # ADAM maintains first and second moment estimates
            # m_t = beta1 * m_{t-1} + (1 - beta1) * gradient
            # v_t = beta2 * v_{t-1} + (1 - beta2) * gradient^2
            # bias_correction_m = m_t / (1 - beta1^t)
            # bias_correction_v = v_t / (1 - beta2^t)
            # update = lr * bias_correction_m / (sqrt(bias_correction_v) + epsilon)
            # sigma = sigma - update
            
            update_adam(scene.volume.sigma, gradients, lr, beta1, beta2, epsilon)
            
        # 7. Update the global sigma_update buffer with optimized values
        # This carries over knowledge to the next rotation angle
        sigma_update = scene.volume.sigma.copy()
        
    return sigma_update
```

**Parameters:**
*   **Learning Rate (LR):** 0.002 [S1, unofficial].
*   **Beta1 ($\beta_1$):** 0.9 [S1, unofficial].
*   **Beta2 ($\beta_2$):** 0.99 [S1, unofficial].
*   **Epsilon ($\epsilon$):** $10^{-8}$ [S1, unofficial].
*   **Nrot:** 40 (covering 360° in 45° steps) [S1, unofficial].
*   **Niter:** 40 iterations per rotation [S1, unofficial].
*   **Resolution:** 256x256 pixels [S1, unofficial].
*   **Sample Count:** 10 samples per pixel [S1, unofficial].
*   **Initial Absorption Coefficient:** 0.001 (for reference) or 0 (for optimization start) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **High GPU Memory Usage:** Auto-differential volume rendering requires significant GPU memory. High resolutions or sample counts cause out-of-memory errors. Mitigation: Use lower resolution (256x256) and sample count (10) [S1, unofficial].
*   **Noise in Reference Images:** Noise in forward-rendered reference images propagates into the optimized volume, causing "noise clouds" [S1, unofficial].
*   **Edge Artifacts:** Undesired clouds around volume edges may be caused by interpolation algorithms or reference image noise [S1, unofficial].
*   **Loss Spikes at Rotation Start:** The MSE loss function shows descending spikes at the start of each new 45° rotation during the first full 360° because the full geometry is not yet known to the optimizer [S1, unofficial].
*   **Convergence Threshold:** The loss function converges around an MSE of 0.0010 after the first full rotation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is iterative and computationally expensive. Total reconstruction time can exceed 15 hours for 40 rotations and 40 iterations per rotation on an Nvidia RTX 3090 [S1, unofficial]. The mean time per 45° rotation is approximately 23.4 minutes [S1, unofficial].
*   **Memory Complexity:** High GPU memory usage (>90% utilization) due to the storage of gradients and intermediate rendering states for auto-differentiation [S1, unofficial]. This limits the feasible resolution and sample count.
*   **Performance:** On an AMD Ryzen 9 3950X and Nvidia RTX 3090, the algorithm is slow but produces reliable results. Only a fraction of CPU cores are utilized, as the bottleneck is GPU rendering and differentiation [S1, unofficial].
*   **Strengths:** Works without pre-trained models; leverages existing differentiable rendering engines (e.g., Mitsuba 2); provides a proof-of-concept for inverse rendering in CT [S1, unofficial].
*   **Limitations:** High memory consumption limits resolution; noisy reference images degrade reconstruction quality; slow convergence; sensitive to initial geometric assumptions during early rotations [S1, unofficial].
*   **When to Use:** Suitable for proof-of-concept demonstrations, small-scale volumetric reconstruction where high precision is not critical, or when traditional reconstruction methods fail due to complex lighting/material interactions that can be modeled in a differentiable renderer.
*   **When Not to Use:** For large-scale clinical CT scans requiring high resolution and speed; when GPU memory is constrained; when reference images are extremely noisy.

## Background
The method was implemented as part of a bachelor's thesis by Vucenovic in 2022 at TU Wien [S1, unofficial]. It builds upon the work of Zhao et al. [ZJL20] for mathematical insights into differentiable volume rendering and Hochhauser et al. [Hoc22] for X-ray spectrum rendering. The implementation specifically uses Mitsuba 2 [NDVZJ19] and a specific fork (master @f4bc83c68f) [S1, unofficial]. The approach is part of the broader field of differentiable rendering, which seeks to make rendering pipelines differentiable with respect to scene parameters, enabling inverse problems like shape and material estimation [S1, unofficial].

## Sources
[S1] Differential Rendering for Computed Tomography, https://www.cg.tuwien.ac.at/research/publications/2022/Vucenovic_2022/Vucenovic_2022-Bachelor%20thesis.pdf, fetched 2026-10-27 (unofficial)
