# NeRDi


## Overview
NeRDi is a single-view synthesis method that leverages a pre-trained 2D image diffusion model to provide general image priors, enabling high-quality novel view synthesis from a single input image [S1, S2]. Unlike traditional NeRF variants that rely on multi-view consistency or explicit geometric supervision alone, NeRDi minimizes the diffusion loss of rendered arbitrary views under input view constraints [S2]. It distinguishes itself by introducing semantic guidance via language embeddings and a geometric loss based on estimated depth maps to regularize the underlying 3D structure [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input image $I_{in}$ of a scene.
*   **Output:** A trained NeRF model $F_\Theta$ capable of rendering novel views.

**Data Structures:**
The method builds upon the standard NeRF representation defined in [Specific Model Architectures](8585-specific-model-architectures.md).
*   **NeRF Model $F_\Theta$:** A differentiable function mapping 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$.
*   **Pre-trained Diffusion Model:** A text-to-image diffusion model (e.g., Stable Diffusion) used as a fixed prior.
*   **Visual Language Model:** A pre-trained model (e.g., CLIP) used to extract text embeddings from the input image.
*   **Depth Estimator:** A monocular depth estimation network used to predict depth maps from rendered views.

**Parameters:**
*   **Learning Rate:** Not given in the sources.
*   **Diffusion Steps:** Not given in the sources.
*   **Frequency Bands ($L$):** As defined in [Specific Model Architectures](8585-specific-model-architectures.md).

**Steps:**
1.  **Semantic Guidance Extraction:** Given the input image $I_{in}$, a pre-trained visual language model extracts two language guides:
    *   $s_0$: An image title containing overall semantics.
    *   $s^*$: A text embedding capturing visual clues from the input image [S1].
2.  **NeRF Initialization:** Initialize a standard NeRF model $F_\Theta$ using the architecture described in [Specific Model Architectures](8585-specific-model-architectures.md).
3.  **Ray Sampling and Rendering:** For each training iteration:
    *   Sample rays $\mathbf{r}$ from the camera pose.
    *   Sample points $\mathbf{x}_i$ along the rays.
    *   Compute encoded positions $\gamma(\mathbf{x}_i)$ and directions $\gamma(\mathbf{d})$.
    *   Query the NeRF to obtain density $\sigma_i$ and color $\mathbf{c}_i$.
    *   Render the pixel color $C(\mathbf{r})$ using volumetric rendering:
        $$
        C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
        $$
4.  **Diffusion Loss Calculation:**
    *   Pass the rendered image $C(\mathbf{r})$ and the semantic guides $s_0, s^*$ into the pre-trained diffusion model.
    *   Compute the diffusion loss $\mathcal{L}_{diff}$ by minimizing the difference between the predicted noise and the actual noise added to the rendered image, conditioned on $s_0$ and $s^*$ [S1, S2]. This step enforces that the rendered view is consistent with the 2D image prior and the semantic content of the input.
5.  **Geometric Loss Calculation:**
    *   Estimate the depth map $D_{est}$ from the rendered view $C(\mathbf{r})$ using a monocular depth estimator.
    *   Compute the geometric loss $\mathcal{L}_{geo}$ based on the consistency between the estimated depth and the NeRF's geometric structure (e.g., depth smoothness or consistency with input constraints) [S1, S2].
6.  **Optimization:**
    *   Update the NeRF parameters $\Theta$ by minimizing the total loss:
        $$
        \mathcal{L}_{total} = \mathcal{L}_{diff} + \lambda \mathcal{L}_{geo}
        $$
        where $\lambda$ is a weighting factor (value not given in sources) [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_NeRDi(input_image, diffusion_model, language_model, depth_estimator, num_iterations):
    # Initialize NeRF model
    nerf_model = initialize_NeRF()
    
    # Extract semantic guidance
    s0 = language_model.get_image_title(input_image)
    s_star = language_model.get_text_embedding(input_image)
    
    for iteration in range(num_iterations):
        # Sample rays and render
        rays = sample_rays(camera_pose)
        x_i, d_i = sample_points(rays)
        sigma_i, c_i = nerf_model.query(x_i, d_i)
        rendered_image = volumetric_render(sigma_i, c_i, rays)
        
        # Compute Diffusion Loss
        # The diffusion model predicts noise given the image and text conditions
        noise_pred = diffusion_model.predict_noise(rendered_image, s0, s_star)
        # Assuming noise was added to the rendered image during training
        # L_diff is the MSE between predicted noise and actual noise
        L_diff = compute_diffusion_loss(noise_pred, rendered_image)
        
        # Compute Geometric Loss
        depth_map = depth_estimator.predict(rendered_image)
        L_geo = compute_geometric_loss(depth_map, nerf_model)
        
        # Total Loss
        L_total = L_diff + lambda_weight * L_geo
        
        # Update NeRF parameters
        nerf_model.update_parameters(L_total)
        
    return nerf_model
```

**Edge Cases and Pitfalls:**
*   **Single View Ambiguity:** Without multi-view constraints, the NeRF may converge to incorrect geometries. The geometric loss and diffusion prior are critical to mitigate this [S1, S2].
*   **Diffusion Model Conditioning:** The quality of semantic guidance depends on the accuracy of the visual language model in extracting relevant features from the single input image [S1].
*   **Depth Estimation Errors:** Monocular depth estimators can be inaccurate for textureless regions or ambiguous scenes, potentially leading to incorrect geometric regularization [S1].

## Subtopics

### Semantic Guidance
Uses language embeddings to guide the diffusion model, improving multi-view consistency. It is chosen when semantic understanding of the scene is beneficial. See [NeRDi](11365-nerdi.md).

### Geometric Regularization
Uses estimated depth maps to constrain the NeRF's geometry. It is chosen when explicit geometric priors are needed to resolve single-view ambiguity. See [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md).

## Operating characteristics
*   **Time Complexity:** Training time is significantly higher than standard NeRF due to the iterative evaluation of the diffusion model and depth estimator for every iteration. Not given in the sources.
*   **Memory Complexity:** Requires storing the pre-trained diffusion model, visual language model, and depth estimator in addition to the NeRF model. Not given in the sources.
*   **Measured Performance:** On the DTU dataset, NeRDi achieves a PSNR that is twice as high as traditional NeRF [S1]. It is suitable for zero-shot NeRF synthesis [S1].
*   **Strengths:** High-quality novel view synthesis from a single image; leverages strong 2D priors; improves multi-view consistency.
*   **Limitations:** High computational cost; reliance on pre-trained models; potential for geometric inaccuracies if depth estimation fails.
*   **When to use:** Single-view synthesis tasks where high visual fidelity is required and multi-view data is unavailable.
*   **When not to use:** Real-time applications or scenarios with limited computational resources.

## Background
NeRDi was proposed in 2023 by Deng et al. [S2]. It was developed to address the limitations of NeRF in single-view synthesis, where traditional methods struggle with ambiguity and lack of geometric constraints. The method builds on the idea of using 2D diffusion models as powerful priors for 3D reconstruction [S2]. It is used in applications requiring high-quality novel view synthesis from sparse or single views, such as virtual reality content creation and 3D asset generation.

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
