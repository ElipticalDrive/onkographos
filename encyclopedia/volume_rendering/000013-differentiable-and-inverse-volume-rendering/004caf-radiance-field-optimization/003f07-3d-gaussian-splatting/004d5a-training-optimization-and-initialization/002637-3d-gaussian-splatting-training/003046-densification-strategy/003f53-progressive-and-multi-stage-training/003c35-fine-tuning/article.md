# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning is the optimization phase in 3D Gaussian Splatting (3DGS) where the initial set of Gaussian primitives is refined to minimize rendering error, often following a coarse initialization or feed-forward prediction. It distinguishes itself from pure initialization methods by actively optimizing geometry and color attributes while dynamically managing the Gaussian population through growing and pruning [S1]. Unlike static representations, this process allows the model to adapt to input data, reducing redundancy and improving photorealism [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An initial set of 3D Gaussian primitives $\{G_i\}$, where each primitive $i$ is defined by mean position $\mathbf{\mu}_i$, covariance matrix $\mathbf{\Sigma}_i$ (or scale/rotation parameters), opacity $\alpha_i$, and spherical harmonic (SH) coefficients $\mathbf{c}_{i,lm}$ for color. These are typically generated via a feed-forward process or Structure-from-Motion (SfM) points [S1].
*   **Output:** Optimized Gaussian parameters $\{\mathbf{\mu}_i, \mathbf{\Sigma}_i, \alpha_i, \mathbf{c}_{i,lm}\}$ that minimize the difference between rendered images and ground truth views [S1].

**Data Structures:**
The core data structure is a list of Gaussian primitives. Each primitive stores:
*   $\mathbf{\mu}_i \in \mathbb{R}^3$: 3D position.
*   $\mathbf{s}_i \in \mathbb{R}^3$: Scale (log-space).
*   $\mathbf{q}_i \in \mathbb{R}^4$: Rotation (quaternion).
*   $\alpha_i \in [0, 1]$: Opacity.
*   $\mathbf{c}_{i,lm}$: SH coefficients for color (typically degree 3 or 4).

**Numbered Steps:**
1.  **Initialization:** Predict or extract an initial set of 3D Gaussian primitives. In feed-forward fine-tuning, a network directly generates geometry and color attributes for initialization [S1].
2.  **Optimization Loop:** Optimize all geometry and color attributes using gradient descent (e.g., Adam) following the vanilla 3DGS differentiable rasterization pipeline [S1].
3.  **Densification Control:** Enable growing (splitting/cloning) and pruning of 3D primitives during the fine-tuning phase [S1]. This is critical because initial feed-forward Gaussians are often redundant [S1].
4.  **Pruning:** Remove Gaussians that contribute negligibly to the scene, significantly reducing the total number of Gaussians and memory consumption while maintaining fidelity [S1].

**Formulas:**
The optimization minimizes a loss function $\mathcal{L}$, typically composed of L1 loss and SSIM loss, potentially with entropy regularization:
$$ \mathcal{L} = (1 - \lambda_{ssim}) \mathcal{L}_{L1} + \lambda_{ssim} \mathcal{L}_{SSIM} + \lambda_e \mathcal{L}_{entropy} $$
Where $\lambda_{ssim}$ and $\lambda_e$ are weighting hyperparameters [S1].

**Parameters:**
*   Optimizer: Adam [S1].
*   Learning Rate: $1 \times 10^{-3}$ [S1].
*   Libraries: `torchsparse` for sparse convolution, `gsplat` for Gaussian rasterization [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** Initial feed-forward Gaussians are often redundant; failing to prune them leads to high memory usage without quality gains [S1].
*   **Convergence:** Pretrained priors accelerate training and enable faster convergence compared to test-time optimization from scratch [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize Gaussians from feed-forward prediction or SfM
gaussians = initialize_gaussians(input_data)

# Configure optimizer
optimizer = Adam(gaussians.parameters(), lr=1e-3)

# Training loop
for step in range(total_steps):
    # Sample a random camera view
    camera = sample_camera_view()
    
    # Render image using differentiable rasterization
    rendered_image = render(gaussians, camera)
    
    # Get ground truth image
    gt_image = get_ground_truth(camera)
    
    # Compute loss
    loss = compute_loss(rendered_image, gt_image)
    
    # Backpropagate
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # Perform densification (growing/pruning) periodically
    if step % densification_interval == 0:
        gaussians = densify_and_prune(gaussians, camera, loss)
```

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that uses an exponentially ascending gradient threshold to regulate splitting and cloning. It allows aggressive densification when few Gaussians exist and restricts it as the scene becomes dense, solving the trade-off between early-stage convergence speed and late-stage overfitting.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Identifies "hard Gaussians"—defective primitives causing blurring or needle-like artifacts—and grows them to recover missing details. It uses multi-view significant positional gradients and rendering error analysis to detect high-error regions, refining the distribution during training to resolve artifacts missed by standard adaptive density control.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
A learning-based, feed-forward module that refines initial 3DGS representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles. It applies a point transformer network to predict attribute residuals, ensuring geometric consistency while recovering details lost in OOD views, unlike standard 3DGS which overfits to input views.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
Reformulates 3DGS by treating Gaussians as MCMC samples from an underlying probability distribution. It converts updates into Stochastic Gradient Langevin Dynamics (SGLD) by introducing noise, removing the need for manual densification heuristics like cloning and opacity resetting.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
A real-time deblurring framework that reconstructs sharp scenes from blurry inputs by manipulating covariance matrices and means using a small MLP during training. It handles defocus and motion blur via differentiable rasterization, addressing sparse point clouds from blurry inputs through KNN interpolation and depth-based pruning.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique compensating for the sparsity of 3D Gaussians in defocus-blurry environments where SfM extraction is poor. It supplements the initial point cloud with synthetic points to ensure adequate coverage in regions where SfM fails, improving fidelity in large-depth-of-field scenarios.

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
Reduces the number of active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter. It addresses excessive memory requirements by masking out unessential Gaussians throughout the entire process, considering both geometric volume and transparency, unlike original 3DGS which stops densifying in the middle of training.

### [Error-Guided Gaussian Reallocation](13012-error-guided-gaussian-reallocation.md)
Redistributes a fixed budget of Gaussian primitives toward regions with persistent coarse-scale reconstruction errors without increasing the total number of Gaussians. It moves low-contribution Gaussians to high-error regions using contribution- and error-aware selection, stabilizing via adaptive relocation count and parameter reset.

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
A post-enhancement framework that improves 3DGS by adding Gaussian kernels with view-dependent opacity to explicitly model specular components. It solves the problem that standard SH cannot decouple diffuse and specular radiance, using distinct primitives for each component to avoid memory and stability issues of high-order SH.

### [Frequency Regularization for Densification](13484-frequency-regularization-for-densification.md)
Addresses over-reconstruction by minimizing the discrepancy between the frequency spectrum of the rendered image and the ground truth in Fourier space. This increases pixel gradients within over-reconstruction regions, enabling a coarse-to-fine densification process guided by progressive frequency annealing.

### [Visibility-Based Pruning for 3D Gaussian Splatting](14019-visibility-based-pruning-for-3d-gaussian-splatting.md)
Removes occluded 3D Gaussian kernels during optimization to create a compact, 'hard' geometry representation. It uses explicit geometric occlusion checks rather than gradient magnitude thresholds to determine which primitives to discard, concentrating alpha values into remaining kernels.

### ACO-EGR Framework
A unified framework integrating Adaptive-Scale Color Offset (ACO) and Error-Guided Gaussian Reallocation (EGR) into a multi-scale 3DGS backbone. It jointly addresses scale-dependent color mismatch and primitive-allocation imbalance by adapting per-Gaussian color across scales and redistributing Gaussians toward high-error regions, improving multi-scale rendering quality without modifying the inference-stage rendering process [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and the number of training steps. Fine-tuning with densification/pruning adds overhead per step but reduces $N$ over time, potentially speeding up later stages [S1].
*   **Memory Complexity:** Initial feed-forward methods may start with redundant Gaussians, requiring significant memory. Effective pruning reduces this to a compact representation [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods, leveraging pretrained weights for faster convergence [S1].
*   **Strengths:** Reduces memory consumption while maintaining high fidelity; accelerates training via pretrained priors [S1].
*   **Limitations:** Requires careful tuning of densification and pruning thresholds to avoid under- or over-reconstruction.
*   **When to use:** When starting from a coarse or feed-forward initialization that contains redundant primitives, or when memory efficiency is a priority [S1].
*   **When not to use:** When the initial initialization is already high-quality and dense, additional fine-tuning with densification may not yield significant benefits and adds computational cost.

## Background
Fine-tuning in the context of 3D Gaussian Splatting has evolved alongside the method itself. The original 3DGS paper introduced adaptive density control during training [S1]. Recent works like EVolSplat have integrated fine-tuning with feed-forward initialization, demonstrating that optimizing geometry and color attributes directly, combined with growing and pruning, can achieve high fidelity with reduced memory usage [S1]. The ACO-EGR framework further extends this by addressing multi-scale rendering issues through joint color and allocation adaptation [S2].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
