# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning is a training optimization stage in 3D Gaussian Splatting (3DGS) that refines an initial set of Gaussian primitives—often generated via a feed-forward network or coarse initialization—by optimizing all geometry and color attributes directly [S1]. It sets itself apart from standard initialization-only approaches by enabling dynamic growing and pruning of primitives during the optimization phase, which significantly reduces memory consumption while maintaining or surpassing photorealism [S1]. This method leverages pretrained priors to accelerate convergence and improve fidelity compared to test-time optimization methods that lack such initialization [S1].

## How it works
**Inputs and Outputs:**
- **Input:** An initial set of 3D Gaussian primitives $\mathcal{G}_{init}$ with geometry (mean $\mu$, covariance $\Sigma$, opacity $\alpha$) and color attributes (spherical harmonics coefficients $c$), typically predicted by a feed-forward process [S1].
- **Output:** A refined set of 3D Gaussian primitives $\mathcal{G}_{final}$ with optimized attributes [S1].

**Data Structures:**
The data structure follows the standard 3DGS representation:
- **Mean ($\mu$):** 3D vector $(x, y, z)$ [S1].
- **Covariance ($\Sigma$):** 3D vector representing rotation and scale, decomposed into a rotation quaternion $q$ and a scale vector $s$ [S1].
- **Opacity ($\alpha$):** Scalar in $[0, 1]$ [S1].
- **Color ($c$):** Spherical Harmonics (SH) coefficients [S1].

**Numbered Steps:**
1. **Initialization:** Predict a set of 3D Gaussian primitives for initialization via a direct feed-forward process where both geometry and color attributes are generated [S1].
2. **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate (typically $1 \times 10^{-3}$) [S1].
3. **Attribute Optimization:** Optimize all geometry and color attributes directly following the vanilla 3DGS rendering and loss pipeline [S1].
4. **Dynamic Management:** Enable growing (densification) and pruning of 3D primitives during the fine-tuning phase [S1].
5. **Convergence:** Continue optimization until convergence, leveraging pretrained priors for faster convergence compared to methods without such priors [S1].

**Formulas:**
The optimization minimizes a loss function $\mathcal{L}$, which typically includes the SSIM loss and an entropy regularization term as used in the EVolSplat implementation:
$$ \mathcal{L} = \lambda_{ssim} \mathcal{L}_{ssim} + \lambda_e \mathcal{L}_{entropy} $$
where $\lambda_{ssim}$ and $\lambda_e$ are weighting factors [S1].

**Parameters:**
- **Learning Rate:** $1 \times 10^{-3}$ (Adam optimizer) [S1].
- **Library:** `gsplat` for Gaussian rasterization [S1].
- **Sparse Convolution:** `torchsparse` if 3D sparse convolution is used [S1].

**Edge Cases and Pitfalls:**
- **Redundancy:** The initial feed-forward Gaussians are often redundant, leading to a significant reduction in the number of Gaussians during fine-tuning [S1]. Pruning is necessary to manage this [S1].
- **Convergence Speed:** Pretrained priors accelerate training; without them, convergence may be slower [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize Gaussians from feed-forward prediction
gaussians = feed_forward_predict_geometry_and_color(input_data)

# Setup optimizer
optimizer = Adam(gaussians.parameters(), lr=1e-3)

# Fine-tuning loop
for step in range(num_steps):
    # Render image from current Gaussians
    rendered_image = gaussian_rasterization(gaussians, camera_params)
    
    # Compute loss (SSIM + Entropy)
    loss = lambda_ssim * ssim_loss(rendered_image, ground_truth) + \
           lambda_entropy * entropy_loss(gaussians)
    
    # Backpropagate and update
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # Enable growing and pruning during fine-tuning
    if step % densification_interval == 0:
        gaussians = densify_and_prune(gaussians, loss_gradients)
```
Note: The specific densification and pruning criteria are not detailed in the source [S1], but the mechanism is enabled [S1].

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism for 3D Gaussian Splatting that uses an exponentially ascending gradient threshold to regulate the splitting and cloning of Gaussians during training. It solves the trade-off between early-stage convergence speed and late-stage overfitting by allowing aggressive densification when few Gaussians exist and restricting it as the scene becomes dense. It sets itself apart from static threshold methods by dynamically adjusting the sensitivity of densification decisions based on the normalized training progress.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
SplatFormer is a learning-based, feed-forward 3D neural module that refines initial 3D Gaussian Splatting (3DGS) representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles [1, unofficial]. It addresses the bias of standard 3DGS optimization, which overfits to input views and fails to generalize to unseen high-elevation perspectives, by applying a point transformer network that predicts attribute residuals [1, unofficial]. Unlike 2D image restoration methods that suffer from multi-view inconsistencies, SplatFormer operates directly on 3D primitives, ensuring geometric consistency while recovering details lost in OOD views [1, unofficial].

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
This method reformulates 3D Gaussian Splatting (3DGS) by treating the set of 3D Gaussians as random samples drawn from an underlying probability distribution describing the scene, specifically as Markov Chain Monte Carlo (MCMC) samples. It converts conventional 3DGS updates into Stochastic Gradient Langevin Dynamics (SGLD) updates by introducing a noise term to promote sample exploration, thereby removing the need for manual densification heuristics like cloning, splitting, and opacity resetting. Unlike sibling methods that rely on complex gradient thresholds or hard constraints, this approach relies on probabilistic state transitions and regularization to manage Gaussian population.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
Deblurring 3D Gaussian Splatting is a real-time deblurring framework that reconstructs sharp 3D scenes from blurry input images by manipulating the covariance matrices and means of 3D Gaussians using a small Multi-Layer Perceptron (MLP) during training. Unlike volumetric rendering-based methods like Deblur-NeRF which require seconds per image, this method uses differentiable rasterization to achieve real-time inference speeds (>800 FPS) while handling both defocus and camera motion blur. It addresses the sparse point clouds caused by blurry inputs through K-nearest-neighbor interpolation and depth-based pruning.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique designed to compensate for the sparsity of 3D Gaussians at the far end of scenes in defocus-blurry environments, where Structure-from-Motion (SfM) extraction is typically poor. It supplements the initial point cloud with synthetic points to ensure adequate coverage in regions where SfM fails to extract features, thereby improving reconstruction fidelity in large-depth-of-field scenarios.

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
Augmented Radiance Field is a post-enhancement framework that improves 3D Gaussian Splatting (3DGS) scenes by adding Gaussian kernels with view-dependent opacity to explicitly model specular components. It solves the problem that standard spherical harmonics (SH) cannot decouple diffuse and specular radiance, leading to poor reconstruction of sharp highlights and material artifacts. Unlike standard 3DGS which assigns SH coefficients to every primitive, this method uses distinct primitives for diffuse and specular components, avoiding the memory and stability issues of high-order SH.

### [Frequency Regularization for Densification](13484-frequency-regularization-for-densification.md)
Frequency Regularization for Densification (FreGS) addresses the over-reconstruction issue in 3D Gaussian Splatting, where high-variance image regions are covered by a few large Gaussians, leading to blur and artifacts. Unlike standard 3D-GS which relies solely on pixel-wise L1 loss in the spatial domain, FreGS minimizes the discrepancy between the frequency spectrum of the rendered image and the ground truth in the Fourier space. This approach significantly increases pixel gradients within over-reconstruction regions, enabling a coarse-to-fine densification process guided by progressive frequency annealing.

### [Error-Guided Redistribution](14037-error-guided-redistribution.md)
Error-Guided Redistribution is a family of training optimization methods for 3D Gaussian Splatting (3DGS) that refine Gaussian allocation based on reconstruction quality metrics. It addresses artifacts caused by suboptimal primitive placement or size by either identifying defective Gaussians for targeted densification or reallocating a fixed budget of primitives toward high-error regions. It differs from standard densification by using explicit error signals (positional gradients or SSIM) rather than averaged gradient magnitudes, and from simple pruning by actively moving primitives to improve fidelity rather than just removing them.

### [CoherentGS](14072-coherentgs.md)
CoherentGS is a regularized optimization approach for sparse novel view synthesis that introduces coherency to an explicit 3D Gaussian representation by constraining Gaussian movement in 2D image space, initialized with monocular depth estimates. It addresses the instability and "needle-like" artifacts of standard 3D Gaussian Splatting (3DGS) when trained on extremely sparse inputs (e.g., 3–4 images) by enforcing single-view and multi-view constraints, preventing overfitting and allowing for the identification of occluded regions suitable for inpainting. Unlike NeRF-based sparse methods that rely on implicit MLP coherency and produce blurry occluded regions, CoherentGS constrains explicit Gaussians in 2D image space, enabling sharper details and structured hallucination via diffusion models.

### [Pruning and Sparsity Control](14431-pruning-and-sparsity-control.md)
Pruning and Sparsity Control methods reduce the number of active 3D Gaussians in a radiance field to lower memory consumption and storage requirements while maintaining rendering quality. Unlike standard 3DGS which relies on opacity thresholds and stops densifying mid-training, these methods actively eliminate redundant primitives throughout the optimization process [12710-gaussian-volume-mask.md]. They are distinguished from sibling methods like [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md) by using explicit geometric or volume-based criteria rather than gradient magnitude alone [14019-visibility-based-pruning-for-3d-gaussian-splatting.md].

### [Rendering-Free Pruning via Hessian Importance](14447-rendering-free-pruning-via-hessian-importance.md)
Rendering-Free Pruning via Hessian Importance (REFINE) is a post-processing framework for 3D Gaussian Splatting (3DGS) that identifies and removes unimportant Gaussian primitives without performing forward rendering passes. It quantifies primitive importance using an analytically approximated Hessian field, which estimates the image-space quality degradation caused by removing a primitive based on parameter-space gradients. Unlike rendering-based methods such as PUP [14431-pruning-and-sparsity-control.md] or LightGaussian, REFINE bypasses rasterization entirely, relying on closed-form algebraic operations to achieve a 3,000× reduction in computational complexity and a ~20× speedup in device latency.

### [GoF-based Dynamic 3DGS Video Construction](14770-gof-based-dynamic-3dgs-video-construction.md)
This method constructs dynamic 3D Gaussian Splatting (3DGS) video by temporally segmenting input sequences into Groups of Frames (GoFs) and independently training Gaussian deformation fields within each segment. It solves the problem of motion drift and geometric inconsistencies that occur when modeling long-range temporal dependencies with a single global deformation field. By limiting the motion degree within short segments, it prevents the deformation field from struggling to adapt to complex, long-sequence dynamics. It sets itself apart from global optimization approaches by decoupling static and dynamic components via weighted MLP parameters and separating foreground/background content to reduce training complexity.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Significantly reduced compared to the initial feed-forward prediction due to pruning of redundant Gaussians [S1].
- **Measured Performance:** Achieves photorealism on par with or surpassing other methods [S1].
- **Scaling:** Not given in the sources.
- **Strengths:** Accelerates network training, enables faster convergence, reduces memory consumption, maintains high fidelity [S1].
- **Limitations:** Relies on the quality of the initial feed-forward prediction [S1].
- **When to use:** When pretrained priors are available and memory efficiency is a concern [S1].
- **When not:** When no pretrained initialization is available or when the initial prediction is poor [S1].

## Background
Fine-tuning in the context of 3D Gaussian Splatting was introduced in EVolSplat for efficient volume-based Gaussian Splatting for urban view synthesis [S1]. The method leverages pretrained weights to accelerate training and improve convergence [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
