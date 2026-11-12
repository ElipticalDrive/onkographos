# Fine-tuning

Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning is the final optimization stage in 3D Gaussian Splatting (3DGS) training pipelines, designed to refine the geometry and color attributes of an initial set of Gaussian primitives to achieve photorealistic novel view synthesis [S1, S2]. It builds on [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md) by applying standard differentiable rendering optimization to a pre-initialized or densified point cloud [S1]. Unlike the coarse stages that focus on global structure and density control, fine-tuning targets high-frequency detail recovery and color accuracy, often leveraging pretrained priors or feed-forward predictions to accelerate convergence [S1]. It sets itself apart from sibling methods like [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) by typically operating on a fixed or dynamically pruned set of primitives rather than introducing new structural hierarchies, and from [Training Objective and Densification](7448-training-objective-and-densification.md) by focusing on attribute refinement rather than population growth [S1, S2].

## How it works
The fine-tuning stage transforms an initial representation of the scene into a high-fidelity model by optimizing all Gaussian parameters simultaneously.

**Inputs and Outputs:**
*   **Input:** An initial set of $N$ 3D Gaussian primitives $\{G_i\}_{i=1}^N$, where each primitive $i$ is defined by mean position $\mu_i$, covariance matrix $\Sigma_i$ (or scale $s_i$ and rotation $q_i$), opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_{i,lm}$ [S1, S2]. These primitives are typically generated via Structure-from-Motion (SfM), a feed-forward network, or a previous densification stage [S1].
*   **Output:** Optimized parameters $\{\mu_i, s_i, q_i, \alpha_i, \mathbf{c}_{i,lm}\}_{i=1}^N$ that minimize the rendering error against a set of training images [S1].

**Data Structures:**
The Gaussian primitives are stored in a flat array or list. Each element contains:
*   `mean`: 3D vector $(x, y, z)$ [S1].
*   `scale`: 3D vector $(s_x, s_y, s_z)$ [S1].
*   `rotation`: Quaternion $(w, x, y, z)$ [S1].
*   `opacity`: Scalar $\alpha \in [0, 1]$ [S1].
*   `sh_coeffs`: Array of spherical harmonic coefficients, typically up to degree 3 (9 coefficients per RGB channel) [S1].

**Numbered Steps:**
1.  **Initialization:** Generate the initial set of 3D Gaussian primitives. This can be done via a direct feed-forward process where geometry and color are predicted simultaneously, or by inheriting the output of a densification strategy [S1]. If using a feed-forward initializer, the model predicts redundant primitives that will be pruned later [S1].
2.  **Optimization Loop:** For a predefined number of iterations or until convergence:
    a. **Sampling:** Randomly sample a training image and its corresponding camera pose [S1].
    b. **Rasterization:** Render the image using differentiable Gaussian splatting. For each pixel, sort the overlapping Gaussians by depth, and compute the color $C$ using alpha compositing:
       $$ C = \sum_{i \in \mathcal{O}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
       where $\mathcal{O}$ is the set of sorted opaque Gaussians, $c_i$ is the color of Gaussian $i$ evaluated from its SH coefficients at the view direction, and $\alpha_i$ is its opacity [S1, S2].
    c. **Loss Computation:** Compute the photometric loss between the rendered image $I_{rendered}$ and the ground truth image $I_{gt}$. The total loss $\mathcal{L}$ is typically a weighted sum:
       $$ \mathcal{L} = \lambda_1 \mathcal{L}_{\text{L1}} + \lambda_2 \mathcal{L}_{\text{D-SSIM}} + \lambda_e \mathcal{L}_{\text{entropy}} $$
       where $\mathcal{L}_{\text{L1}}$ is the L1 loss, $\mathcal{L}_{\text{D-SSIM}}$ is the D-SSIM loss, and $\mathcal{L}_{\text{entropy}}$ is an entropy loss to encourage opacity values to be either 0 or 1 [S1].
    d. **Gradient Update:** Update all parameters ($\mu, s, q, \alpha, \mathbf{c}$) using an optimizer, typically Adam with a learning rate of $1 \times 10^{-3}$ [S1].
3.  **Pruning (Optional but Common):** During fine-tuning, if the initial set contains redundant primitives (e.g., from a feed-forward initializer), prune Gaussians with low opacity or small volume to reduce memory consumption while maintaining fidelity [S1]. This step may be interleaved with optimization or performed at the end.
4.  **Convergence Check:** Stop when the validation loss stabilizes or the maximum iteration count is reached.

**Parameters:**
*   **Learning Rate:** $1 \times 10^{-3}$ for Adam optimizer [S1].
*   **Loss Weights:** $\lambda_1, \lambda_2, \lambda_e$ are typically set to balance L1, SSIM, and entropy terms, though exact values are not specified in the sources [S1].
*   **SH Degree:** Typically degree 3 for high-fidelity color [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** If initialized via feed-forward methods, the initial number of Gaussians may be significantly larger than necessary, leading to high memory usage during the early fine-tuning steps [S1]. Pruning is essential to mitigate this [S1].
*   **Overfitting:** Fine-tuning can overfit to the training views if the number of iterations is excessive or if regularization is insufficient [S1].
*   **Precision:** Standard floating-point precision is used; no specific bit-packing is mentioned for the fine-tuning stage itself [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fine_tune_gaussians(gaussians, training_images, camera_poses, num_iterations):
    """
    Optimizes 3D Gaussian parameters for photorealistic rendering.
    
    Args:
        gaussians: List of Gaussian primitives, each with mean, scale, rotation, opacity, sh_coeffs.
        training_images: List of ground truth images.
        camera_poses: List of camera poses corresponding to training_images.
        num_iterations: Number of optimization steps.
        
    Returns:
        Optimized gaussians.
    """
    optimizer = Adam(gaussians.parameters(), lr=1e-3)
    
    for step in range(num_iterations):
        # Sample a random training image
        idx = random.randint(0, len(training_images) - 1)
        gt_image = training_images[idx]
        camera_pose = camera_poses[idx]
        
        # Render the scene
        rendered_image = render(gaussians, camera_pose)
        
        # Compute loss
        l1_loss = compute_l1_loss(rendered_image, gt_image)
        ssim_loss = compute_dssim_loss(rendered_image, gt_image)
        entropy_loss = compute_entropy_loss(gaussians)
        
        # Weighted sum (weights not specified in sources, assumed standard)
        loss = 0.8 * l1_loss + 0.2 * ssim_loss + 0.001 * entropy_loss
        
        # Backpropagate and update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Optional: Prune low-opacity Gaussians if initialized from feed-forward
        if step % 1000 == 0:
            prune_gaussians(gaussians, opacity_threshold=0.005)
            
    return gaussians
```

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism for 3D Gaussian Splatting that uses an exponentially ascending gradient threshold to regulate the splitting and cloning of Gaussians during training. It solves the trade-off between early-stage convergence speed and late-stage overfitting by allowing aggressive densification when few Gaussians exist and restricting it as the scene becomes dense. It sets itself apart from static threshold methods by dynamically adjusting the sensitivity of densification decisions based on the normalized training progress.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Hard Gaussian Splatting (HGS) is a densification strategy for 3D Gaussian Splatting that identifies "hard Gaussians"—defective primitives causing blurring or needle-like artifacts—and grows them to recover missing scene details. Unlike standard 3DGS, which relies on averaged view-space positional gradients that smooth out local errors and hinder the growth of defective Gaussians, HGS uses multi-view significant positional gradients (PGHGS) and rendering error analysis (REHGS) to detect regions with high reconstruction error. It builds on [Fine-tuning](9138-fine-tuning.md) by refining the Gaussian distribution during the training phase to resolve artifacts that standard adaptive density control misses.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
SplatFormer is a learning-based, feed-forward 3D neural module that refines initial 3D Gaussian Splatting (3DGS) representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles. It addresses the bias of standard 3DGS optimization, which overfits to input views and fails to generalize to unseen high-elevation perspectives, by applying a point transformer network that predicts attribute residuals. Unlike 2D image restoration methods that suffer from multi-view inconsistencies, SplatFormer operates directly on 3D primitives, ensuring geometric consistency while recovering details lost in OOD views.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
This method reformulates 3D Gaussian Splatting (3DGS) by treating the set of 3D Gaussians as random samples drawn from an underlying probability distribution describing the scene, specifically as Markov Chain Monte Carlo (MCMC) samples. It converts conventional 3DGS updates into Stochastic Gradient Langevin Dynamics (SGLD) updates by introducing a noise term to promote sample exploration, thereby removing the need for manual densification heuristics like cloning, splitting, and opacity resetting. Unlike sibling methods that rely on complex gradient thresholds or hard constraints, this approach relies on probabilistic state transitions and regularization to manage Gaussian population.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
Deblurring 3D Gaussian Splatting is a real-time deblurring framework that reconstructs sharp 3D scenes from blurry input images by manipulating the covariance matrices and means of 3D Gaussians using a small Multi-Layer Perceptron (MLP) during training. Unlike volumetric rendering-based methods like Deblur-NeRF which require seconds per image, this method uses differentiable rasterization to achieve real-time inference speeds (>800 FPS) while handling both defocus and camera motion blur. It addresses the sparse point clouds caused by blurry inputs through K-nearest-neighbor interpolation and depth-based pruning.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique designed to compensate for the sparsity of 3D Gaussians at the far end of scenes in defocus-blurry environments, where Structure-from-Motion (SfM) extraction is typically poor. It supplements the initial point cloud with synthetic points to ensure adequate coverage in regions where SfM fails to extract features, thereby improving reconstruction fidelity in large-depth-of-field scenarios.

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
Gaussian Volume Mask is a densification strategy for 3D Gaussian Splatting that reduces the number of active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter. It addresses the excessive memory and storage requirements caused by the proliferation of Gaussians during training, which often include redundant primitives with negligible volume or transparency. Unlike original 3DGS, which relies solely on opacity-based control and stops densifying in the middle of training, this method consistently masks out unessential Gaussians throughout the entire process by considering both geometric volume and transparency.

### [Error-Guided Gaussian Reallocation](13012-error-guided-gaussian-reallocation.md)
Error-Guided Gaussian Reallocation (EGR) is a training-stage module that redistributes a fixed budget of Gaussian primitives toward regions with persistent coarse-scale reconstruction errors, without increasing the total number of Gaussians. It refines post-densification Gaussian allocation by moving low-contribution (LC) Gaussians to high-error (HE) regions within a fixed Gaussian set, using contribution- and error-aware selection. Unlike methods using explicit rollback or separate re-correction modules, EGR uses adaptive relocation count, parameter reset, and continued optimization as its stabilization strategy.

### ACO-EGR Framework
ACO-EGR is a unified framework that integrates ACO and EGR into a multi-scale 3DGS backbone to jointly address scale-dependent color mismatch and primitive-allocation imbalance [S2]. The framework augments a multi-scale 3DGS backbone with two training-stage modules: ACO and EGR [S2]. Unlike existing methods that keep fixed color and spatial allocation after densification, ACO-EGR adapts both Gaussian color and primitive allocation together [S2]. ACO-EGR improves rendering quality without modifying the inference-stage rendering process itself; it only changes the training/optimization of the Gaussian parameters [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians $N$ and the number of pixels $P$ per iteration, i.e., $O(N \cdot P)$, due to the rasterization process [S1]. The number of iterations is typically fixed or determined by convergence, often ranging from tens of thousands to hundreds of thousands depending on the scene complexity [S1].
*   **Memory Complexity:** Linear with respect to the number of Gaussians $N$, storing mean, scale, rotation, opacity, and SH coefficients for each [S1]. Fine-tuning can reduce memory usage if pruning is applied to remove redundant primitives from initial feed-forward predictions [S1].
*   **Measured Performance:** EVolSplat, which uses fine-tuning, achieves photorealism on par with or surpassing other methods, leveraging pretrained weights to accelerate training and enable faster convergence compared to test-time optimization methods [S1].
*   **Scaling:** Fine-tuning scales well with the number of Gaussians, but the memory footprint can become a bottleneck for very large scenes if pruning is not employed [S1].
*   **Strengths:** High photorealism, fast convergence when using pretrained initializations, and compatibility with standard differentiable rendering pipelines [S1].
*   **Limitations:** Can overfit to training views if not regularized; initial redundancy from feed-forward methods can increase early-stage memory usage [S1].
*   **When to Use:** When high-fidelity novel view synthesis is required and a good initial point cloud or feed-forward prediction is available [S1].
*   **When Not to Use:** When computational resources are extremely limited and pruning cannot be applied, or when the initial initialization is of very poor quality [S1].

## Background
Fine-tuning in 3D Gaussian Splatting originated as the final stage of the training pipeline introduced in the original 3DGS paper [15] (cited in [S1]). It was designed to refine the sparse point cloud obtained from Structure-from-Motion into a dense, photorealistic representation. Recent developments, such as EVolSplat, have integrated fine-tuning with feed-forward initialization to accelerate convergence and reduce memory usage [S1]. The ACO-EGR framework further extends fine-tuning by addressing multi-scale rendering issues through adaptive color offsets and error-guided reallocation [S2].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
