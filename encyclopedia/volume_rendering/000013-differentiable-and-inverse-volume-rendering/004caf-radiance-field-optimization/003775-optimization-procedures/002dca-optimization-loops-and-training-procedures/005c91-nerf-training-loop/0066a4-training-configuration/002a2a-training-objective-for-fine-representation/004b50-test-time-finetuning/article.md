# Test-Time Finetuning
Also called: Code Finetuning, Guidance-Finetuning

## Overview
Test-Time Finetuning is a post-sampling optimization step that refines a sampled scene code by minimizing a combined loss of rendering fidelity and a diffusion prior, while keeping the model parameters frozen [S1]. It addresses the limitation where reconstruction guidance alone cannot strictly enforce rendering constraints towards faithful reconstruction [S1]. It sets itself apart from two-stage generative neural field methods by retaining the prior term during the optimization of the scene code, avoiding the biased and noisy scene codes that result from ignoring this term [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sampled scene code $x$ initialized from the sampling process (e.g., DDIM step) [S1].
*   **Output:** An optimized scene code $x$ that balances rendering accuracy and diffusion prior consistency [S1].
*   **Frozen Parameters:** The diffusion model parameters and the decoder parameters are kept fixed during this step [S1].

**Data Structures:**
*   **Scene Code ($x$):** The latent representation of the scene being optimized. The specific layout of $x$ depends on the underlying representation (e.g., triplane, voxel grid), but it is treated as a tensor of learnable parameters for this optimization step [S1].

**Steps:**
1.  Initialize the scene code $x$ from the sampling process [S1].
2.  Optimize $x$ using the Adam optimizer to minimize the combined loss function [S1].
3.  The optimization minimizes the following objective:
    $$ \min_x \lambda_{\text{rend}} L_{\text{rend}}(x) + \lambda'_{\text{diff}} L_{\text{diff}}(x) $$
    where:
    *   $L_{\text{rend}}(x)$ is the rendering loss, measuring the difference between rendered images and ground truth [S1].
    *   $L_{\text{diff}}(x)$ is the diffusion prior loss, enforcing consistency with the learned data distribution [S1].
    *   $\lambda_{\text{rend}}$ is the rendering loss weight [S1].
    *   $\lambda'_{\text{diff}}$ is the test-time prior weight [S1].

**Parameters:**
*   $\lambda'_{\text{diff}}$: The test-time prior weight. It should be lower than the training weight $\lambda_{\text{diff}}$ because the prior learned from the training dataset is less reliable when transferred to a different testing dataset [S1]. The exact value is not specified in the sources [S1].
*   Optimizer: Adam [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_finetuning(scene_code_x, ground_truth_images, diffusion_model, decoder, lambda_rend, lambda_diff_prime):
    """
    Finetunes the scene code x by minimizing rendering and diffusion losses
    while keeping model parameters frozen.
    
    Args:
        scene_code_x: Tensor representing the scene code, requires gradients.
        ground_truth_images: Tensor of ground truth images for rendering loss.
        diffusion_model: Frozen diffusion model for prior loss.
        decoder: Frozen decoder to render images from scene code.
        lambda_rend: Weight for rendering loss.
        lambda_diff_prime: Weight for diffusion prior loss at test time.
        
    Returns:
        Optimized scene_code_x.
    """
    # Initialize optimizer for the scene code only
    optimizer = Adam([scene_code_x])
    
    # Optimization loop
    # Note: The number of iterations is not specified in the sources [S1]
    for step in range(num_iterations):
        optimizer.zero_grad()
        
        # 1. Compute Rendering Loss
        # Render images from the current scene code using the frozen decoder
        rendered_images = decoder(scene_code_x)
        # Calculate rendering loss (e.g., L2 or L1 between rendered and ground truth)
        # The specific loss function L_rend is not detailed, but standard reconstruction losses are implied [S1]
        loss_rend = compute_rendering_loss(rendered_images, ground_truth_images)
        
        # 2. Compute Diffusion Prior Loss
        # The diffusion loss L_diff is computed based on the scene code x
        # This typically involves the diffusion model's prediction of noise or score
        # The exact implementation of L_diff is not detailed, but it relies on the frozen diffusion model [S1]
        loss_diff = compute_diffusion_loss(scene_code_x, diffusion_model)
        
        # 3. Combine Losses
        total_loss = lambda_rend * loss_rend + lambda_diff_prime * loss_diff
        
        # 4. Backpropagate and Update
        total_loss.backward()
        optimizer.step()
        
    return scene_code_x
```

**Edge Cases and Pitfalls:**
*   **Prior Reliability:** Using the training prior weight $\lambda_{\text{diff}}$ at test time can lead to suboptimal results because the prior is less reliable for unseen data distributions [S1].
*   **Frozen Parameters:** If the decoder or diffusion model parameters are not properly frozen, the optimization may inadvertently update the model, leading to incorrect results [S1].

## Subtopics

### [Warm-up Strategy](14795-warm-up-strategy.md)
A training phase that prevents the model from overfitting to the input view and ensures fine details are captured during distillation by temporarily freezing or adjusting objectives before full joint training. It serves as a stabilization mechanism for radiance-field optimization, particularly when introducing perceptual or adversarial losses that might otherwise cause mode collapse or weak distillation if applied immediately.

### [Frozen Latent Code](18270-frozen-latent-code.md)
Frozen Latent Code is a finetuning variant in differentiable volume rendering where the per-identity latent code is kept fixed while only the model parameters are optimized. It solves the problem of adapting a pretrained volumetric prior to a target identity by reducing the optimization degrees of freedom compared to joint optimization. Unlike [Model Fitting (Fine-tuning)](18287-model-fitting-fine-tuning.md), which jointly optimizes the latent code and model weights, this method optimizes only the model parameters, yielding visually similar results but with different quantitative metrics.

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization iterations and the cost of computing the rendering and diffusion losses per iteration. The sources do not provide specific complexity measures [S1].
*   **Memory Complexity:** Requires memory for the scene code $x$ and the gradients during optimization. The frozen model parameters also consume memory but are not updated [S1].
*   **Performance:** Improves reconstruction fidelity by enforcing the diffusion prior during the test phase [S1].
*   **Strengths:** Addresses the issue of reconstruction guidance alone not strictly enforcing rendering constraints [S1]. Avoids the biased and noisy scene codes associated with two-stage methods that ignore the prior term [S1].
*   **Limitations:** The reliability of the diffusion prior decreases when transferred to different testing datasets, necessitating a lower test-time prior weight [S1].
*   **When to Use:** When faithful reconstruction is required and the scene code has been sampled from a diffusion process [S1].
*   **When Not to Use:** When computational resources are extremely limited and the overhead of additional optimization steps is prohibitive [S1].

## Background
Test-Time Finetuning is introduced in the context of Single-Stage Diffusion NeRF, a unified approach to 3D generation and reconstruction [S1]. It builds upon the observation that previous two-stage methods often ignore the prior term during the training of auto-decoders, leading to biased scene codes [S1]. The method was proposed to address these limitations by integrating the prior term into the test-time optimization of the scene code [S1].

## Sources
[S1] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
