# Differential Rendering for CT Reconstruction

Also called: Differentiable Rendering for CT Reconstruction, Differentiable Volume Rendering Reconstruction, Gradient-based Volumetric Reconstruction, Inverse Rendering for CT, Iterative Differential CT

## Overview
Differential Rendering for CT Reconstruction is an inverse rendering technique that reconstructs the volumetric absorption coefficients of an object from a series of 2D X-ray projection images by optimizing scene parameters via gradient descent [S1, unofficial]. It formulates Computed Tomography (CT) as an iterative optimization problem where the 3D volume is treated as a differentiable parameter space, allowing the use of auto-differentiable rendering engines to compute gradients of a projection loss [S1, unofficial]. Unlike traditional algebraic or analytical methods, this approach relies on end-to-end differentiability of the forward projection, enabling the use of modern optimizers like ADAM without explicit derivation of the back-projection operator [S1, unofficial]. It is distinct from neural implicit methods (e.g., [HoloINR](4745-holoinr.md)) by typically operating on explicit voxel grids rather than continuous neural fields, and from deep learning approaches by requiring no pre-trained models [S1, unofficial].

## How it works
The method minimizes the Mean Squared Error (MSE) between reference projection images (ground truth) and differentially rendered images of a current volume estimate. The volume is represented as a grid of absorption coefficients, and the optimization updates these coefficients to match the observed projections from multiple angles.

### Inputs and Outputs
*   **Inputs:**
    *   Reference projection images $I_{ref}$ acquired at various rotation angles.
    *   Camera/Source geometry (positions and orientations corresponding to the reference images).
    *   Scene parameters $\pi$, including the initial absorption coefficient matrix $\sigma$ and rotation state.
*   **Outputs:**
    *   Optimized absorption coefficient matrix $\sigma$ representing the reconstructed 3D volume.

### Data Structures
The volume is represented as a 3D voxel grid. In the reference implementation, the grid size is $16 \times 16 \times 16$ [S1, unofficial].
*   **Voxel Value:** Represents the absorption coefficient $\sigma$.
    *   $\sigma = 0$: No absorption.
    *   $\sigma = 1$: Fully absorbing.
    *   Typical initial non-zero value for object voxels: $10^{-4}$ [S1, unofficial].
*   **Scene Parameters $\pi$:** Contains the current rotation angle and the volume data $\sigma$.

### Formulas
The core objective is to minimize the loss $\mathcal{L}$ between the rendered image $I$ and the reference image $I_{ref}$:
$$
\mathcal{L} = \text{MSE}(I, I_{ref}) = \frac{1}{N} \sum_{k=1}^{N} (I_k - I_{ref, k})^2
$$
where $N$ is the number of pixels.

The gradient of the loss with respect to the absorption coefficient matrix $\pi_0$ (denoted as $\sigma$ in implementation) is computed via the chain rule through the differentiable renderer [S1, unofficial]:
$$
\frac{\partial \mathcal{L}}{\partial \pi_0} = \frac{\partial}{\partial \pi_0} \iint f(x, y; \pi) \, dx \, dy = \frac{\partial I_i}{\partial \pi_0}
$$
where $f(x, y; \pi)$ is the rendering function mapping scene parameters to pixel values, and $I_i$ is the pixel value [S1, unofficial].

### Parameters
*   **Optimizer:** ADAM [KB15].
    *   Learning Rate (LR): $0.002$ [S1, unofficial].
    *   $\beta_1$ (first moment average): $0.9$ [S1, unofficial].
    *   $\beta_2$ (second moment average): $0.99$ [S1, unofficial].
    *   $\epsilon$ (numerical stability): $10^{-8}$ [S1, unofficial].
*   **Rotation Schedule:**
    *   $N_{rot}$: Number of $45^\circ$ steps. Typical value: $40$ (covering $360^\circ$) [S1, unofficial].
    *   Rotation angle per step $i$: $(i \times 45.0) \mod 360$ [S1, unofficial].
*   **Iterations:**
    *   $N_{iter}$: Optimization iterations per rotation. Typical value: $40$ [S1, unofficial].
*   **Rendering Settings:**
    *   Resolution: $256 \times 256$ pixels [S1, unofficial].
    *   Samples per pixel: $10$ [S1, unofficial].
*   **Initialization:**
    *   Initial absorption coefficient: $0.001$ (or $0$ for the update matrix) [S1, unofficial].

### Pseudocode
The following algorithm outlines the reconstruction process. It assumes a differentiable rendering engine `render_differential` that returns both the image and the gradient computation graph, and an optimizer `adam_step` that updates parameters based on gradients.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differential_ct_reconstruction(Nrot, Niter, initial_sigma, camera_geometry):
    """
    Reconstructs a 3D volume from 2D projections using differential rendering.
    
    Args:
        Nrot (int): Number of 45-degree rotation steps.
        Niter (int): Number of optimization iterations per rotation.
        initial_sigma (array): Initial 3D absorption coefficient grid.
        camera_geometry (list): List of camera poses corresponding to reference images.
        
    Returns:
        array: Optimized absorption coefficient grid.
    """
    # Initialize the update matrix for the volume parameters
    # In the first run, this is zero-filled; subsequent runs carry over optimized values
    sigma_update = None 
    
    # Total iterations for convergence monitoring
    total_iterations = Nrot * Niter
    
    for i in range(Nrot):
        # 1. Calculate current rotation angle
        rotation_angle = (i * 45.0) % 360.0
        
        # 2. Setup scene with current rotation and volume parameters
        # The scene includes the volume defined by (initial_sigma + sigma_update)
        scene = create_scene(
            volume_data=initial_sigma + sigma_update,
            rotation=rotation_angle,
            camera_poses=camera_geometry
        )
        
        # 3. Generate reference image (Ground Truth)
        # This is typically done once per angle if using measured data, 
        # or forward rendered if synthesizing.
        image_ref = render_forward(scene)
        
        # 4. Initialize sigma_update if null (first rotation)
        if sigma_update is None:
            sigma_update = np.zeros_like(initial_sigma)
            
        # 5. Optimization Loop for current rotation
        for j in range(Niter):
            # Render the scene with current parameters (differentiable)
            # This computes the forward projection and sets up the gradient graph
            image_current = render_differential(scene)
            
            # Calculate Loss (MSE)
            loss = mse(image_current, image_ref)
            
            # Backpropagate to compute gradients w.r.t scene parameters (sigma)
            loss.backward()
            
            # Update parameters using ADAM optimizer
            # The optimizer updates the underlying tensor 'sigma' in the scene
            adam_step(lr=0.002, beta1=0.9, beta2=0.99, eps=1e-8)
            
        # 6. After Niter iterations, extract the optimized volume data
        # This becomes the starting point for the next rotation
        sigma_update = scene.volume.sigma.data
        
    return sigma_update
```

### Edge Cases and Pitfalls
*   **GPU Memory Constraints:** Auto-differentiable volume rendering is memory-intensive. High resolutions or sample counts can exceed GPU memory limits (e.g., >90% usage on RTX 3090) [S1, unofficial]. Mitigation involves using lower resolutions ($256 \times 256$) and sample counts (10) [S1, unofficial].
*   **Noise Propagation:** Noise in the reference images (from forward rendering or measurement) propagates into the optimized volume, creating "noise clouds" [S1, unofficial].
*   **Edge Artifacts:** Undesired artifacts around volume edges may arise from interpolation algorithms in the renderer or reference image noise [S1, unofficial].
*   **Loss Spikes:** At the start of each new $45^\circ$ rotation during the first full $360^\circ$ cycle, the MSE loss exhibits descending spikes. This occurs because the optimizer has not yet seen the full geometry from all angles, leading to incorrect geometric assumptions [S1, unofficial].
*   **Convergence:** The loss function typically converges to an MSE of approximately $0.0010$ after the first full rotation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is iterative. For $N_{rot}$ rotations and $N_{iter}$ iterations per rotation, the total number of forward/backward passes is $N_{rot} \times N_{iter}$. In the reference implementation, $40 \times 40 = 1600$ iterations were performed, taking approximately 15.6 hours on an AMD Ryzen 9 3950X and Nvidia RTX 3090 [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the differentiable rendering engine's computation graph and the volume grid. It remains high and constant during optimization (>90% GPU memory) [S1, unofficial].
*   **Strengths:**
    *   Does not require pre-trained deep learning models [S1, unofficial].
    *   Leverages existing auto-differentiable rendering engines (e.g., Mitsuba 2) [S1, unofficial].
    *   Provides a flexible framework for incorporating different physical models or regularizers via the loss function.
*   **Limitations:**
    *   Computationally expensive (hours for low-resolution reconstructions) [S1, unofficial].
    *   Limited by GPU memory, restricting resolution and sample counts, which leads to noisy results [S1, unofficial].
    *   Sensitive to noise in reference images [S1, unofficial].

## Background
This method was implemented and evaluated in a 2022 bachelor's thesis by Vucenovic at TU Wien, utilizing the Mitsuba 2 differentiable rendering framework [S1, unofficial]. It builds upon the theoretical foundations of differentiable rendering for inverse problems, specifically referencing the work of Zhao et al. [ZJL20] for the mathematical insights into differentiating the rendering function with respect to absorption coefficients. The approach is part of a broader trend of using differentiable simulators for inverse problems in computer graphics and medical imaging, offering an alternative to traditional iterative reconstruction techniques like SIRT [3101-sirt-simultaneous-iterative-reconstruction-technique.md] or ART [6944-algebraic-reconstruction-technique-art.md] by bypassing the need for explicit Jacobian calculations.

## Sources
[S1] Differential Rendering for Computed Tomography, https://www.cg.tuwien.ac.at/research/publications/2022/Vucenovic_2022/Vucenovic_2022-Bachelor%20thesis.pdf, fetched 2026-10-27 (unofficial)
