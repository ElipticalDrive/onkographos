# Optimization Loops and Training Procedures

Also called: Inverse Rendering Loop

## Overview
This family defines the iterative procedures used to minimize the photometric error between rendered images and ground-truth observations by updating the parameters of differentiable volume or radiance fields [S1, unofficial]. It differs from the underlying field representations (e.g., [Neural Radiance Fields](4972-neural-radiance-fields.md), [Plenoxels](5942-plenoxels.md)) by focusing on the optimization dynamics—specifically how rays are sampled, how gradients are propagated, and how hyperparameters like learning rates and regularization are scheduled [G].

## Shared principles
All methods in this family share the core objective of minimizing a photometric loss $\mathcal{L}$ between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$ for a batch of rays $\mathcal{R}$ [G]. The optimization loop generally follows these steps: ray sampling, forward pass, loss computation, backpropagation, and scheduling [G]. The conceptual pseudocode for the inverse problem loop is [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_rendering_loop(theta, samples, targets, optimizer, num_steps):
    """
    Standard inverse rendering optimization loop.
    """
    # 1. Initialization
    theta = initialize_params(theta)
    
    for step in range(num_steps):
        # 2. Forward Pass
        prediction = trace(theta, samples)
        
        # 3. Loss Computation
        loss = distance(prediction, targets)
        
        # 4. Backpropagation
        gradient = backward(loss, theta)
        
        # 5. Update
        theta = optimizer.update(theta, gradient)
        
    return theta
```
Differentiation strategies fall into three main categories: reverse-mode automatic differentiation, the adjoint method, and analytic inversion or explicit gradients [G]. Choosing a faithful model, identifiable parameters, a stable estimator, and a good initialization remains difficult even when the loop structure is simple [S1, unofficial].

## Subtopics

### [NeRF Training Loop](970-nerf-training-loop.md)
Iterative optimization procedure minimizing photometric difference between rendered images and ground-truth views by updating a neural radiance field. Sets itself apart by computing gradients for backpropagation and including adaptive sampling and occupancy grid pruning steps specific to training. Choose for standard NeRF-style reconstruction.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
Hyperparameter setup combining Adam with staircase exponential decay and selective L2 weight decay on MLPs, excluding positional encodings and appearance embeddings. Differs by explicitly masking L2 regularization to prevent divergence and using a decay schedule holding the initial rate for 10,000 steps. Choose when high numerical stability and controlled regularization are required.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
Training procedure for implicit neural representations of 3D volumes minimizing difference between reconstructed volume and ground-truth input. Sets itself apart by operating on volumetric data patches rather than ray-based pixel supervision. Choose for simple volumetric field training without complex regularization.

### [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md)
Stage-based optimization loop alternating between image-based and ray-based sampling modes, applying stochastic preconditioning with exponential alpha decay, and optionally upsampling parameters. Distinguishes itself by explicitly managing memory via ray batching and progressively refining field resolution. Choose for large-scale scenes or hierarchical representations.

### [Photometric Optimization](5790-photometric-optimization.md)
Not given in the sources.

### [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md)
Training procedure decoupling geometric shape and PSF parameter optimization into two sequential sub-problems. Solves computational bias from simultaneous optimization by alternating between approximate narrow-band FT for shape and exact full-frequency FT for PSF. Choose when optical blur or PSF estimation is coupled with geometry reconstruction.

### [Optimization Baselines](6675-optimization-baselines.md)
Classical evolutionary baseline using covariance matrix adaptation (CMA-ES). Choose for establishing performance baselines without gradient-based methods.

### [Generalized Rendering Models](6681-generalized-rendering-models.md)
Optimizes rendering models for consistency across subjects with high inter-patient variations, showing benefits of deep architectures over manual transfer functions. Choose for clinical or cross-subject applications.

### [Two-Stage Inverse Rendering Optimization](7387-two-stage-inverse-rendering-optimization.md)
Two-stage pipeline first optimizing geometry using an approximate differentiable renderer (silhouette-based), then optimizing reflectance using a physically-based differentiable renderer (ray-tracing). Choose when decoupling geometry and material optimization improves convergence.

## Background
The optimization loops described here originate from differentiable rendering research, formalized as an inverse problem where a differentiable renderer sits inside an optimizer [S1, unofficial]. The standard NeRF training loop uses Adam with a specific learning rate schedule and MSE loss [G]. These methods are widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and inverse rendering [G].

## Sources
[S1] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
