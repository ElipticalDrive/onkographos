# GSN Inversion

Also called: Inversion of Locally Conditioned Radiance Fields, Latent Code Optimization

## Overview
GSN Inversion is a method for view synthesis and scene completion that inverts source views into a latent code grid to locally condition a trained radiance field. It solves the problem of generating unseen parts of a scene or editing scenes by optimizing latent codes rather than sampling from the prior [S1]. Unlike methods relying on global coordinate systems, GSN uses a local coordinate system that is robust to rearrangements of latent codes, such as 2D rotations, whereas global systems suffer rendering quality degradation as rotation angles increase [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of source views $S = \{(X, T)_i\}_{i=t-5}^{t}$ (5 previous frames, where $X$ is the image and $T$ is the camera pose) and target views $T = \{(X, T)_i\}_{i=t}^{t+5}$ (camera poses for target views) [S1].
*   **Outputs:** An optimized latent code grid $\hat{W}$ and rendered images $\hat{S}$ (reconstruction of source views) and $\hat{T}$ (prediction of target views) [S1].

**Data Structures:**
*   **Encoder Output ($W_0$):** A latent code grid of shape $R^{c \times s \times s}$, derived from a shared feature volume $V \in R^{c \times s \times s \times s}$ via average pooling over the height dimension [S1].
*   **Radiance Field ($f_{\theta_f}$):** A locally conditioned radiance field with frozen weights $\theta_f$ [S1].

**Parameters:**
*   **Optimization Iterations:** 1000 SGD iterations [S1].
*   **Source Window:** 5 previous frames ($i=t-5$ to $t$) [S1].
*   **Loss Metric:** LPIPS loss [60] for orientation selection [S1].

**Steps:**
1.  **Initialization:** Feed the source views $S$ through the trained encoder $E_{\theta_E}$ to predict an initial latent code grid $W_0$ [S1].
2.  **Canonical Orientation Selection:** Since scenes do not share a canonical orientation and the generator may prefer a specific one, predict $W_0$ at multiple rotation angles of the source camera poses $\{T_i\}_{i=t-5}^{t}$ about the y-axis [S1]. Select the orientation that minimizes an auto-encoding LPIPS loss as the 'preferred orientation' [S1]. Use this preferred orientation for subsequent optimization [S1].
3.  **Latent Code Optimization:** Optimize $W_0$ via SGD for 1000 iterations to obtain $\hat{W}$, while freezing the weights of the radiance field $f_{\theta_f}$ [S1].
4.  **Reconstruction and Prediction:**
    *   Render observations using the camera poses of $S$ to produce $\hat{S}$ (reconstruction) [S1].
    *   Render from the camera poses of target views $T$ to produce $\hat{T}$ (prediction of unseen parts) [S1].

**Training Loss (for Encoder):**
The encoder is trained by minimizing the following reconstruction loss [S1]:
$$
L(\hat{X}, T, W; \theta_E) = \|W - E_{\theta_E}(\hat{X}, T)\|^2 + \|\hat{X} - f(E_{\theta_E}(\hat{X}, T), T)\|^2
$$
where the first term encourages the reconstruction of the local latent grid, and the second term encourages samples from the locally conditioned radiance field $f$ to match the original input views $\hat{X}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gsn_inversion(source_views_S, target_poses_T, encoder_E, radiance_field_f):
    """
    source_views_S: List of tuples (image, pose) for i=t-5 to t
    target_poses_T: List of poses for target views
    encoder_E: Trained encoder E_theta_E
    radiance_field_f: Trained radiance field f_theta_f (weights frozen)
    """
    
    # Step 1: Find preferred orientation
    best_orientation = None
    min_lpips_loss = infinity
    
    # Test multiple y-axis rotations of source poses
    for rotation_angle in candidate_y_axis_rotations:
        rotated_source_poses = rotate_poses_y_axis(source_views_S, rotation_angle)
        
        # Predict initial latent code grid W0
        W0 = encoder_E(source_views_S, rotated_source_poses)
        
        # Compute auto-encoding LPIPS loss
        # Render reconstruction using W0 and rotated source poses
        reconstructed_images = radiance_field_f(W0, rotated_source_poses)
        lpips_loss = compute_lpips_loss(source_views_S.images, reconstructed_images)
        
        if lpips_loss < min_lpips_loss:
            min_lpips_loss = lpips_loss
            best_orientation = rotation_angle
            best_W0 = W0
            best_source_poses = rotated_source_poses

    # Step 2: Optimize latent code grid
    W_opt = best_W0.clone()
    W_opt.requires_grad = True
    
    optimizer = SGD([W_opt], lr=learning_rate)
    
    for iteration in range(1000):
        optimizer.zero_grad()
        
        # Render reconstruction using current W_opt and best orientation poses
        reconstructed_images = radiance_field_f(W_opt, best_source_poses)
        
        # Compute reconstruction loss (L1 or similar, as per standard GAN inversion practices)
        # Note: The specific loss for optimization step 3 is not explicitly defined in S1 beyond "reconstruction loss w.r.t S"
        # We assume a standard pixel-wise loss for reconstruction optimization
        loss = compute_reconstruction_loss(source_views_S.images, reconstructed_images)
        
        loss.backward()
        optimizer.step()
        
    W_hat = W_opt.detach()
    
    # Step 3: Render outputs
    # Reconstruct source views
    S_hat = radiance_field_f(W_hat, best_source_poses)
    
    # Predict target views
    T_hat = radiance_field_f(W_hat, target_poses_T)
    
    return W_hat, S_hat, T_hat
```

## Subtopics

### View Synthesis via GAN Inversion
A method for view synthesis where source views are inverted into the prior to obtain a latent code, which is then used to locally condition the radiance field to render target views [S1]. Given source views $S = \{(X, T)_i\}_{i=t-5}^{t}$ and target views $T = \{(X, T)_i\}_{i=t}^{t+5}$, the source views $S$ are inverted into the prior to obtain a latent code [S1]. The resulting latent code is used to locally condition the radiance field, and observations are rendered using the camera poses of the target views $T$ [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** GSN improves view synthesis quality on Vizdoom and AVD datasets compared to GTM-SM and ISS, outperforming them in both Memorization (reconstruction) and Hallucination (scene completion) metrics (L1 and SSIM) [S1].
*   **Strengths:** Robust to rearrangements of latent codes (e.g., 2D rotation of $W$) due to the local coordinate system [S1].
*   **Limitations:** Scenes do not share a canonical orientation, requiring multiple rotations to find the generator's preferred orientation [S1]. The optimization process freezes the radiance field weights, limiting joint optimization [S1].

## Background
GSN Inversion is based on "Unconstrained Scene Generation with Locally Conditioned Radiance Fields" [S1]. It builds on standard practices for GAN inversion [57] to handle the task of view synthesis and scene completion [S1]. The method was evaluated on Vizdoom and AVD datasets [S1].

## Sources
[S1] Unconstrained Scene Generation with Locally Conditioned Radiance Fields, https://doi.org/10.1109/iccv48922.2021.01404, fetched 2026-10-26
