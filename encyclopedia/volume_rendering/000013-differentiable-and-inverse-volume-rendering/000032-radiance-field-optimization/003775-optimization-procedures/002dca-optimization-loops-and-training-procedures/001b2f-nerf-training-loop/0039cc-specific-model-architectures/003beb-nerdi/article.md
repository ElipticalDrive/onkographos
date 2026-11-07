# NeRDi


## Overview
NeRDi is a single-view synthesis method that leverages a pre-trained 2D image diffusion model to provide general image priors, enabling high-quality novel view synthesis from a single input image [S1, S2]. Unlike standard NeRF which requires multiple views for optimization, NeRDi minimizes the diffusion loss of rendered views under the constraint of the input view, supplemented by semantic and geometric guidance [S1, S2]. It is distinguished by its use of a visual language model to generate conditional inputs for the diffusion process and its integration of estimated depth maps to regularize 3D geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input image $I_{in}$.
*   **Output:** A radiance field $f_\theta(\mathbf{x}, \mathbf{d})$ parameterized by $\theta$, capable of rendering novel views.

**Data Structures:**
*   **Radiance Field:** Standard MLP-based representation as defined in [Specific Model Architectures](8585-specific-model-architectures.md), outputting volume density $\sigma$ and color $\mathbf{c}$.
*   **Visual Language Model (VLM):** Pre-trained model used to extract text embeddings from the input image.
*   **Diffusion Model:** Pre-trained 2D image diffusion model used for loss computation.
*   **Depth Estimator:** Module used to estimate depth maps from the input image.

**Parameters:**
*   $\theta$: Radiance field weights.
*   $s_0$: Image title embedding (overall semantics).
*   $s^*$: Text embedding extracted from the input image (visual clues).
*   Not given in the sources.

**Steps:**
1.  **Semantic Guidance Generation:** Given the input image $I_{in}$, use a pre-trained visual language model to extract two language guides:
    *   $s_0$: An image title containing overall semantics.
    *   $s^*$: A text embedding capturing visual clues from $I_{in}$ [S1].
2.  **Depth Estimation:** Estimate a depth map $D_{est}$ from the input image $I_{in}$ [S1, S2].
3.  **Radiance Field Optimization:** Optimize the radiance field parameters $\theta$ by minimizing a composite loss function. The optimization involves:
    *   **Diffusion Loss:** Render arbitrary views from the current radiance field. Compute the diffusion loss between these rendered views and the pre-trained diffusion model, conditioned on the semantic guides $s_0$ and $s^*$ [S1, S2]. This ensures the rendered views adhere to general image priors.
    *   **Geometric Loss:** Compute a geometric loss based on the estimated depth map $D_{est}$ and the depth derived from the radiance field. This regularizes the underlying 3D geometric structure [S1, S2].
    *   **Input View Constraint:** Ensure the rendered view corresponding to the input camera pose matches the input image $I_{in}$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# NeRDi Optimization Loop
# Inputs: input_image, pretrained_vlm, pretrained_diffusion_model, depth_estimator
# Hyperparameters: learning_rate, num_iterations

# 1. Extract Semantic Guides
s0 = pretrained_vlm.get_image_title(input_image)
s_star = pretrained_vlm.extract_text_embedding(input_image)

# 2. Estimate Depth
depth_est = depth_estimator.estimate(input_image)

# 3. Initialize Radiance Field Parameters theta
theta = initialize_radiance_field()

for iteration in range(num_iterations):
    # Sample a random camera pose for novel view synthesis
    camera_pose = sample_random_pose()
    
    # Render image from current radiance field
    rendered_image = render_radiance_field(theta, camera_pose)
    
    # Compute Diffusion Loss
    # The diffusion model is conditioned on s0 and s_star
    diffusion_loss = compute_diffusion_loss(
        rendered_image, 
        pretrained_diffusion_model, 
        condition_text=[s0, s_star]
    )
    
    # Compute Geometric Loss
    # Estimate depth from the rendered image or radiance field
    rendered_depth = estimate_depth_from_radiance_field(theta, camera_pose)
    geometric_loss = compute_depth_consistency_loss(rendered_depth, depth_est)
    
    # Compute Input View Constraint Loss (if optimizing for the input view specifically)
    # This ensures the model respects the single input image
    input_view_rendered = render_radiance_field(theta, input_camera_pose)
    input_loss = compute_reconstruction_loss(input_view_rendered, input_image)
    
    # Total Loss
    total_loss = diffusion_loss + geometric_loss + input_loss
    
    # Update Radiance Field Parameters
    theta = update_parameters(theta, total_loss, learning_rate)

return theta
```

**Edge Cases and Pitfalls:**
*   **Single View Ambiguity:** Without sufficient geometric or semantic priors, single-view synthesis can suffer from depth ambiguity. The geometric loss based on estimated depth helps mitigate this [S1, S2].
*   **Semantic Mismatch:** If the VLM extracts misleading semantics, the diffusion guidance may produce artifacts. The dual guidance ($s_0$ and $s^*$) aims to balance overall semantics and visual clues [S1].
*   **Depth Estimation Errors:** Inaccurate depth estimation can lead to incorrect geometric regularization. The quality of the depth estimator is critical [S1, S2].

## Subtopics

### Single-View Synthesis
NeRDi is designed for synthesizing novel views from a single image, addressing the limitation of standard NeRF which requires multiple views [S1, S2].

### Diffusion-Based Regularization
It uses a pre-trained 2D diffusion model to enforce image realism and consistency, a key differentiator from other NeRF variants that rely solely on photometric loss [S1, S2].

### Semantic Guidance
The integration of language embeddings from a VLM provides semantic context, improving multi-view consistency [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Training involves iterative optimization of the radiance field and diffusion model interactions.
*   **Memory Complexity:** Not given in the sources. Requires storing the radiance field, diffusion model, and VLM.
*   **Measured Performance:** On the DTU dataset, NeRDi achieves a PSNR twice as high as traditional NeRF [S1].
*   **Strengths:** High-quality single-view synthesis, robust to limited input data, leverages strong 2D priors [S1, S2].
*   **Limitations:** Relies on the quality of pre-trained models (VLM, Diffusion, Depth Estimator). Performance may degrade if the input image is out-of-distribution for the pre-trained models [S1, S2].
*   **When to Use:** When only a single image is available for novel view synthesis and high realism is required [S1, S2].
*   **When Not to Use:** When multiple views are available, standard NeRF or multi-view methods may be more appropriate and potentially more accurate [S2].

## Background
NeRDi was introduced in 2023 by Deng et al. [S2]. It was developed to address the limitations of NeRF in single-view synthesis, leveraging the power of pre-trained 2D diffusion models to provide general image priors [S2]. The method builds on the concept of using diffusion models for inverse graphics and incorporates semantic and geometric guidance to improve 3D consistency [S1, S2]. It is suitable for zero-shot NeRF synthesis [S1].

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
