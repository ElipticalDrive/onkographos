# GAN Inversion Encoder and Optimization
Also called: GSN Inversion Encoder and Optimization

## Overview
This method enables view synthesis and scene completion by inverting a trained generative radiance field (specifically a Generative Scene Network, GSN) into a latent code grid [S1]. It solves the problem of generating unseen parts of a scene or editing scenes by optimizing local latent codes rather than sampling from the prior distribution [S1]. Unlike methods relying on global coordinate systems which suffer rendering quality degradation as rotation angles increase, this approach uses a local coordinate system that is robust to rearrangements of latent codes, such as 2D rotations [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed source views $S = \{(X, T)_i\}_{i=t-5}^t$, where $X$ represents the image data and $T$ represents the camera pose transformations [S1].
*   **Outputs:** An optimized latent code grid $\hat{W}$ that conditions the radiance field to synthesize new views [S1].

**Data Structures:**
*   **Encoder Parameters ($\theta_E$):** Weights of the neural network encoder $E_{\theta_E}$ [S1].
*   **Feature Volume ($V$):** A 3D volume with layout $V \in \mathbb{R}^{c \times s \times s \times s}$, where $c$ is the channel dimension and $s$ is the spatial resolution [S1].
*   **Latent Code Grid ($W_0$):** The initial grid of latent codes with layout $W_0 \in \mathbb{R}^{c \times s \times s}$, derived from $V$ [S1].
*   **Radiance Field ($f$):** The trained generator function that maps latent codes and camera poses to images [S1].

**Numbered Steps:**
1.  **Encoder Architecture Setup:** Construct an encoder $E_{\theta_E}$ consisting of a UNet backbone [47] with a ResNet-50 encoder [S1]. The encoder maps input views and poses to a feature volume: $E_{\theta_E} : \mathbb{R}^{3 \times w \times h} \times SE(3) \rightarrow \mathbb{R}^{c \times s \times s \times s}$ [S1].
2.  **Feature Extraction and Back-projection:** Process the input views $\hat{X}$ through the backbone to obtain feature maps [S1]. Back-project these feature maps using the camera poses $T$ into the shared feature volume $V \in \mathbb{R}^{c \times s \times s \times s}$ [S1].
3.  **Latent Grid Initialization:** Perform average pooling over the height dimension of $V$ to produce the initial latent code grid $W_0 \in \mathbb{R}^{c \times s \times s}$ [S1].
4.  **Orientation Selection:** Since scenes do not share a canonical orientation, predict $W_0$ at multiple rotation angles about the y-axis for the source views $\{T_i\}_{i=t-5}^t$ [S1]. Define the preferred orientation as the one that minimizes an auto-encoding LPIPS loss [60]. Use this orientation during the subsequent optimization [S1].
5.  **Optimization:** Optimize the initial grid $W_0$ via Stochastic Gradient Descent (SGD) for 1000 iterations to obtain the final grid $\hat{W}$ [S1]. The radiance field parameters are frozen during this process [S1].

**Formulas:**
The encoder is trained by minimizing the reconstruction loss [S1]:
$$
L(\hat{X}, T, W; \theta_E) = \|W - E_{\theta_E}(\hat{X}, T)\|_2^2 + \|\hat{X} - f(E_{\theta_E}(\hat{X}, T), T)\|_2^2
$$
Where:
*   The first term $\|W - E_{\theta_E}(\hat{X}, T)\|_2^2$ encourages the reconstruction of the local latent grid [S1].
*   The second term $\|\hat{X} - f(E_{\theta_E}(\hat{X}, T), T)\|_2^2$ encourages samples from the locally conditioned radiance field $f$ to match the original input views [S1].

**Parameters:**
*   **Optimization Iterations:** 1000 iterations [S1].
*   **Optimizer:** SGD [S1].
*   **Source View Window:** 6 views ($i=t-5$ to $t$) [S1].

**Edge Cases and Pitfalls:**
*   **Mixed Precision:** Mixed precision training is not applied to the discriminator (if present in the base GSN) as training stability decreases in that case [S1].
*   **Orientation Ambiguity:** Failure to select the generator's preferred orientation via LPIPS minimization may result in suboptimal reconstruction due to lack of canonical alignment [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gan_inversion_encoder_optimization(source_views, camera_poses, encoder, radiance_field, num_iterations=1000):
    """
    Inverts a GSN for view synthesis.
    
    Args:
        source_views: List of images X.
        camera_poses: List of poses T.
        encoder: Trained encoder E_theta_E.
        radiance_field: Trained generator f.
        
    Returns:
        optimized_W: Optimized latent code grid.
    """
    # Step 1: Find preferred orientation
    best_orientation = None
    min_lpips_loss = infinity
    
    # Test multiple rotation angles about the y-axis
    for angle in rotation_angles:
        rotated_poses = rotate_poses(camera_poses, angle, axis='y')
        # Predict initial latent grid W0 for this orientation
        W0_candidate = encoder(source_views, rotated_poses)
        
        # Calculate auto-encoding LPIPS loss
        # Note: LPIPS calculation requires specific perceptual loss implementation
        lpips_loss = compute_lpips_loss(source_views, radiance_field(W0_candidate, rotated_poses))
        
        if lpips_loss < min_lpips_loss:
            min_lpips_loss = lpips_loss
            best_orientation = angle
            
    # Apply preferred orientation
    final_poses = rotate_poses(camera_poses, best_orientation, axis='y')
    
    # Step 2: Initialize W0 using the encoder with preferred orientation
    W0 = encoder(source_views, final_poses)
    
    # Step 3: Optimize W0 via SGD
    # Note: W0 requires gradient tracking
    W_opt = W0.clone().detach().requires_grad_(True)
    optimizer = SGD([W_opt], lr=learning_rate) # Learning rate not specified in sources [G]
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        # Forward pass through radiance field
        synthesized_views = radiance_field(W_opt, final_poses)
        
        # Compute reconstruction loss
        # Term 1: Latent grid reconstruction (if ground truth W available during training, 
        # but here we optimize for synthesis, so primarily term 2 matters for inference)
        # The source defines L for encoder training. For inference optimization, 
        # the loss is typically just the image reconstruction term against source views.
        # However, the source states "optimize W0 via SGD... to get W_hat".
        # We assume the loss function used for optimization mirrors the second term of Eq 3.
        loss = torch.norm(synthesized_views - source_views, p=2)
        
        loss.backward()
        optimizer.step()
        
    return W_opt
```

## Subtopics

### [GSN Inversion](16716-gsn-inversion.md)
GSN Inversion is a method for view synthesis and scene completion that inverts source views into a latent code grid to locally condition a trained radiance field. It solves the problem of generating unseen parts of a scene or editing scenes by optimizing latent codes rather than sampling from the prior. Unlike methods relying on global coordinate systems, GSN uses a local coordinate system that is robust to rearrangements of latent codes, such as 2D rotations, whereas global systems suffer rendering quality degradation as rotation angles increase.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Robustness to 2D rotations and rearrangements of latent codes due to the local coordinate system [S1].
*   **Limitations:** Requires a pre-trained GSN and encoder [S1]. Sensitive to the choice of canonical orientation [S1].
*   **When to use:** When view synthesis or scene completion is required for a scene generated by a GSN, particularly when the scene orientation is unknown or variable [S1].
*   **When not to use:** When global coordinate consistency is strictly required and the scene is not generated by a GSN [S1].

## Background
The method originates from the work on Unconstrained Scene Generation with Locally Conditioned Radiance Fields [S1]. It adapts common practices for GAN inversion [57] to the context of Generative Scene Networks (GSN). The encoder architecture is conceptually similar to prior works [22, 36] [S1].

## Sources
[S1] Unconstrained Scene Generation with Locally Conditioned Radiance Fields, https://doi.org/10.1109/iccv48922.2021.01404, fetched 2026-10-26
