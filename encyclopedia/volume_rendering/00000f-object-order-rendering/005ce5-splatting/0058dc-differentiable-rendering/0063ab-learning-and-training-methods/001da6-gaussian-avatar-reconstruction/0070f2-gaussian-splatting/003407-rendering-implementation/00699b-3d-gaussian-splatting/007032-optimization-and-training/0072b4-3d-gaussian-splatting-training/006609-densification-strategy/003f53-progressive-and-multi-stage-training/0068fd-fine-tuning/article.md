# Fine-tuning

Also called: 3DGS Fine-tuning, Fine-tuning Strategy, Gaussian Splatting Fine-tuning, Per-scene optimization

## Overview
Fine-tuning in the context of 3D Gaussian Splatting (3DGS) refers to a per-scene optimization process that refines an initial, often coarse or generated, Gaussian representation into a high-fidelity radiance field by minimizing rendering losses against ground-truth images [S1, S3]. It addresses the gap between initial reconstruction (e.g., from SfM points or a generative prior) and the final photorealistic output by allowing the geometry (means, covariances) and appearance (SH coefficients, opacity) to adapt to the specific scene [S3]. It sets itself apart from static initialization methods by actively optimizing all primitive attributes, and from global training strategies by focusing on a single-scene refinement phase, often leveraging pretrained weights or specific initialization schemes to accelerate convergence [S3].

## How it works
The fine-tuning process typically begins with an initial set of 3D Gaussians $\mathcal{G}_0$ derived from a point cloud, a generative model, or a feed-forward network. The core mechanism involves differentiable rasterization to render images from novel or training viewpoints and backpropagating the difference between rendered and ground-truth images to update the Gaussian parameters.

### Inputs and Outputs
- **Inputs**: 
  - Initial Gaussian set $\mathcal{G}_0 = \{\mu_i, \Sigma_i, \alpha_i, \mathbf{c}_i\}_{i=1}^N$, where $\mu_i$ is the mean, $\Sigma_i$ is the covariance, $\alpha_i$ is the opacity, and $\mathbf{c}_i$ are spherical harmonic (SH) coefficients [S3].
  - A set of training images $I_{gt}$ with known camera poses $P_j$ [S1, S3].
  - Loss function $\mathcal{L}$, typically a combination of L1 loss and SSIM [S3].
- **Outputs**: 
  - Optimized Gaussian set $\mathcal{G}^*$ with refined geometry and appearance [S3].

### Data Structures
The Gaussian set $\mathcal{G}$ is stored as a list or array of primitives. Each primitive $i$ contains:
- **Mean ($\mu_i$)**: 3D vector $(x, y, z)$ [S3].
- **Covariance ($\Sigma_i$)**: 3x3 symmetric positive-definite matrix, often parameterized by a 6D rotation vector and 3D scale vector for efficient optimization [S3, G].
- **Opacity ($\alpha_i$)**: Scalar in $[0, 1]$ [S3].
- **SH Coefficients ($\mathbf{c}_i$)**: Vector of coefficients for view-dependent color, typically up to degree 3 (45 coefficients) [S3, G].

### Steps
1. **Initialization**: Generate initial Gaussians. This can be done via:
   - Direct mapping from SfM point clouds [S3].
   - Feed-forward prediction of geometry and color attributes from a pretrained network [S3].
   - Voxel representation and marching cubes for mesh-based initialization [S1].
2. **Rendering**: For each training image $j$ with pose $P_j$:
   - Project 3D Gaussians to 2D screen space.
   - Sort Gaussians by depth.
   - Composite colors using alpha blending: $C_j = \sum_{i \in \mathcal{G}} c_i \alpha_i \prod_{k < i} (1 - \alpha_k)$ [S3, G].
3. **Loss Computation**: Calculate the loss between rendered image $C_j$ and ground truth $I_{gt, j}$:
   $$ \mathcal{L} = \lambda_1 \mathcal{L}_{L1} + \lambda_2 \mathcal{L}_{SSIM} + \lambda_e \mathcal{L}_{entropy} $$
   where $\mathcal{L}_{L1}$ is the L1 distance, $\mathcal{L}_{SSIM}$ is the structural similarity index, and $\mathcal{L}_{entropy}$ is an entropy loss to encourage opacity sparsity [S3].
4. **Optimization**: Update Gaussian parameters using an optimizer (e.g., Adam) [S3]:
   $$ \theta_{t+1} = \theta_t - \eta \nabla_\theta \mathcal{L} $$
   where $\theta$ represents all trainable parameters ($\mu, \Sigma, \alpha, \mathbf{c}$) and $\eta$ is the learning rate [S3].
5. **Densification/Pruning (Optional)**: During fine-tuning, enable growing (splitting/cloning) and pruning of primitives to adapt density to scene complexity [S3]. This is often controlled by gradient magnitude or opacity thresholds [S3, G].
6. **Iteration**: Repeat steps 2-5 for a fixed number of iterations or until convergence [S3].

### Parameters
- **Learning Rate**: Typically $1 \times 10^{-3}$ for Adam optimizer [S3].
- **Loss Weights**: $\lambda_1, \lambda_2, \lambda_e$ are hyperparameters balancing the loss terms [S3].
- **Iterations**: Depends on scene complexity; often fewer iterations than initial training due to pretrained priors [S3].

### Edge Cases and Pitfalls
- **Overfitting**: Fine-tuning on a small number of views can lead to overfitting and artifacts in novel views [S1].
- **Initial Quality**: Poor initialization (e.g., noisy SfM points) can lead to suboptimal local minima [S3].
- **Opacity Artifacts**: High-opacity floaters can attenuate gradients to true surfaces, requiring opacity reset or regularization [S1, G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fine_tune_3dgs(initial_gaussians, training_data, num_iterations, learning_rate):
    """
    Fine-tune 3D Gaussian Splatting representation.
    
    Args:
        initial_gaussians: List of Gaussian primitives (means, covariances, opacities, SH coeffs).
        training_data: List of tuples (image, camera_pose).
        num_iterations: Number of optimization steps.
        learning_rate: Learning rate for Adam optimizer.
        
    Returns:
        optimized_gaussians: Refined Gaussian primitives.
    """
    # Initialize optimizer with all Gaussian parameters
    optimizer = Adam(initial_gaussians.parameters(), lr=learning_rate)
    
    for iteration in range(num_iterations):
        # Sample a random training image and pose
        image, pose = random.choice(training_data)
        
        # Render the scene from the current pose
        rendered_image = render_gaussians(initial_gaussians, pose)
        
        # Compute loss
        l1_loss = L1_loss(rendered_image, image)
        ssim_loss = SSIM_loss(rendered_image, image)
        entropy_loss = Entropy_loss(initial_gaussians.opacities)
        
        # Combined loss (weights are hyperparameters)
        loss = 0.7 * l1_loss + 0.2 * ssim_loss + 0.1 * entropy_loss
        
        # Backpropagate and update parameters
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Optional: Densification and pruning based on gradients
        if iteration % densification_interval == 0:
            initial_gaussians.densify_and_prune(optimizer)
            
    return initial_gaussians
```

## Subtopics

### Image-space Gaussian Fine-tuning
A fine-tuning process that accounts for different visual modes in a volume by generating multiple radiance fields using different transfer functions and optimizing the Gaussian set against rendered images [S4, unofficial]. It selects a series of transfer functions $\{TF_h\}$, where each $TF_h$ is non-zero only within specific value intervals of the physical field $F$, resulting in radiance fields $RF_h = TF_h(F)$ [S4, unofficial]. For each radiance field, 64 images are rendered through volumetric integration to provide supervision data, and the image-space loss is backpropagated to update the Gaussian set $G_h$ [S4, unofficial]. In practice, 5 transfer functions are selected per volume [S4, unofficial]. After fine-tuning, the Gaussians $G_h$ for each mode can be concatenated or separately rendered [S4, unofficial].

### Ada-GS
A baseline method that trains vanilla 3D Gaussian Splatting directly on 2D stylized images [S5]. It preserves better details than GS-Ada but requires training from scratch for every new style image, limiting practical usability compared to methods like GSS [S5].

## Operating characteristics
- **Time Complexity**: Depends on the number of Gaussians $N$, the number of training images $M$, and the number of iterations $T$. Rendering is $O(N \log N)$ due to sorting, and optimization is $O(T \cdot M \cdot N)$ [S3, G].
- **Memory Complexity**: $O(N \cdot D)$, where $D$ is the dimensionality of Gaussian parameters (means, covariances, SH coefficients) [S3, G].
- **Performance**: Fine-tuning with pretrained priors can accelerate convergence and reduce memory consumption compared to training from scratch [S3].
- **Strengths**: High fidelity, ability to refine coarse initializations, flexibility in handling different initialization schemes [S3].
- **Limitations**: Sensitive to initialization quality, risk of overfitting on sparse views, computational cost of densification [S3, G].
- **When to use**: When a high-quality initial reconstruction is available but needs refinement, or when using generative priors that require per-scene adaptation [S1, S3].
- **When not to use**: When real-time inference is critical and pre-training is not feasible, or when the initial reconstruction is extremely poor [S3].

## Background
Fine-tuning in 3DGS is inspired by techniques in NeRF and CNNs. In NeRF, methods like LOLNeRF and Code-NeRF use per-scene optimization to refine implicit representations [S1]. In CNNs, fine-tuning involves fixing lower layers and optimizing higher layers for specific tasks [S2]. The application of fine-tuning to 3DGS allows for efficient adaptation of Gaussian primitives to specific scenes, leveraging the differentiable nature of Gaussian splatting [S3].

## Sources
[S1] GPN: Generative Point-Based NeRF, https://doi.org/10.1007/978-3-031-82021-2_20, fetched 2026-10-30
[S2] CNNs Based Viewpoint Estimation for Volume Visualization, https://doi.org/10.1145/3309993, fetched 2026-10-29
[S3] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S4] Variable Basis Mapping for Real-Time Volumetric Visualization, https://doi.org/10.48550/arxiv.2601.09417, fetched 2026-10-26 (unofficial)
[S5] Gaussian Splatting in Style, https://doi.org/10.1007/978-3-031-85187-2_15, fetched 2026-10-26
