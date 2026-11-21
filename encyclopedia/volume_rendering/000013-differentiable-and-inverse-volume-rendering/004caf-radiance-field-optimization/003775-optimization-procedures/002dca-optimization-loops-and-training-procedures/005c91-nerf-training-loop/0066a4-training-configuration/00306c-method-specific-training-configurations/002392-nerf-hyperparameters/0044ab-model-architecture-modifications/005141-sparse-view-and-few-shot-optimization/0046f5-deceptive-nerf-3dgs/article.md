# Deceptive-NeRF/3DGS

Also called: Deceptive-3DGS, DeceptiveX

## Overview
Deceptive-NeRF/3DGS is a sparse-view reconstruction method that densifies limited input observations by generating photorealistic pseudo-observations via a 2D diffusion model, treating them as real training views to overcome overfitting and artifacts inherent in few-view settings [S1]. Unlike sibling methods that use diffusion models as per-step regularizers (e.g., SDS) or scorers during optimization, this method infers the diffusion model only during a dedicated generation phase, drastically reducing computational cost while improving reconstruction quality [S1]. It builds on [Sparse View and Few-Shot Optimization](14428-sparse-view-and-few-shot-optimization.md) by actively expanding the training dataset rather than relying solely on implicit regularization.

## How it works
The method alternates between training a coarse scene representation, generating pseudo-observations guided by an uncertainty measure, and retraining the representation on the combined set of real and pseudo-observations. This cycle is repeated to progressively densify the input views.

### Inputs and Outputs
**Inputs:**
*   A set of $W$ sparse input images $I = \{I_1, I_2, \dots, I_W\}$ with known calibrated camera poses [S1].
*   A 2D pre-trained diffusion model [S1].
*   A base radiance field representation $F_\Theta$ (either an MLP-based NeRF or 3D Gaussian Splatting) [S1].

**Outputs:**
*   A refined radiance field representation $F_\Theta$ capable of rendering high-quality novel views and depth maps [S1].

### Data Structures
The method relies on the underlying data structures of the chosen backbone:
1.  **NeRF Backbone (Deceptive-NeRF):** Uses Nerfacto from NerfStudio [S1]. This involves an MLP parameterizing the mapping $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is spatial position, $\mathbf{d} \in \mathbb{R}^3$ is viewing direction, $\sigma$ is volume density, and $\mathbf{c}$ is view-dependent color [S1]. The rendering uses standard volume rendering equations with proposal sampling, scene contraction, and appearance embeddings [S1].
2.  **3DGS Backbone (Deceptive-3DGS):** Uses Splatfacto from NerfStudio [S1]. This involves a set of 3D Gaussians. For Deceptive-3DGS, Gaussians are randomly initialized without utilizing Colmap point clouds [S1]. A bounding box is defined using the outermost input cameras to constrain initialization [S1]. The rendering computes pixel color as $\hat{\mathbf{C}} = \sum_{i=1}^n \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1}(1-\alpha_j)$, where $\mathbf{c}_i$ is color from SH coefficients, and $\alpha_i$ is opacity derived from a 2D Gaussian multiplied by an opacity logit $o$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deceptive_nerf_3dgs(input_images, camera_poses, diffusion_model, num_iterations=5):
    """
    Main training loop for Deceptive-NeRF/3DGS.
    
    Args:
        input_images: List of W sparse input images.
        camera_poses: Corresponding camera poses for input_images.
        diffusion_model: Pre-trained 2D diffusion model.
        num_iterations: Number of alternation steps (default 5).
        
    Returns:
        Trained radiance field representation.
    """
    # 1. Initialize Scene Representation
    # For NeRF: Initialize MLP parameters theta
    # For 3DGS: Randomly initialize Gaussians within bounding box defined by outermost cameras
    scene_rep = initialize_scene(input_images, camera_poses)
    
    # 2. Initial Coarse Training
    # Train scene_rep on input_images to get a coarse reconstruction
    scene_rep = train_coarse(scene_rep, input_images, camera_poses)
    
    # 3. Iterative Densification Loop
    for iteration in range(num_iterations):
        # Step A: Render Coarse Views
        # Sample novel viewpoints within the bounding box defined by outermost input cameras
        novel_viewpoints = sample_novel_viewpoints(camera_poses)
        
        # Render RGB images and Depth maps from current scene_rep
        coarse_rgb_images, coarse_depth_maps = render(scene_rep, novel_viewpoints)
        
        # Step B: Compute Uncertainty Measure
        # Calculate uncertainty map to guide diffusion generation
        # High uncertainty corresponds to artifacts in coarse rendering
        uncertainty_maps = compute_uncertainty(coarse_rgb_images, coarse_depth_maps)
        
        # Step C: Generate Pseudo-Observations
        # Feed coarse images into deceptive diffusion model guided by uncertainty
        # The diffusion model enhances RGB-D images to generate high-quality pseudo-observations
        pseudo_observations = generate_pseudo_observations(
            coarse_rgb_images, 
            coarse_depth_maps, 
            uncertainty_maps, 
            diffusion_model
        )
        
        # Step D: Update Training Set
        # Combine real input images and newly generated pseudo-observations
        # The method increases the number of training views by 5 to 10 times total
        current_training_set = input_images + pseudo_observations
        current_training_poses = camera_poses + get_poses(pseudo_observations)
        
        # Step E: Retrain Scene Representation
        # Optimize scene_rep minimizing photometric loss on both real and pseudo images
        scene_rep = train_on_combined(
            scene_rep, 
            current_training_set, 
            current_training_poses
        )
        
    return scene_rep

def compute_uncertainty(rgb_images, depth_maps):
    """
    Computes an uncertainty measure to guide the diffusion model.
    Areas with high uncertainty (brighter in map) correspond to artifacts.
    """
    # Implementation detail: [G]
    # The source describes the uncertainty measure conceptually but does not provide
    # the exact mathematical formula for its computation. It is used to guide
    # the diffusion model to remove artifacts.
    # A typical implementation would involve analyzing variance in depth or
    # consistency between multi-view renders, but the specific metric is not defined.
    # We assume a placeholder function that returns a map of same dimensions as RGB.
    return generate_uncertainty_map(rgb_images, depth_maps)

def generate_pseudo_observations(rgb, depth, uncertainty, diffusion_model):
    """
    Uses the deceptive diffusion model to generate pseudo-observations.
    """
    # The diffusion model takes the coarse RGB-D images and the uncertainty map
    # as guidance to generate photorealistic pseudo-observations.
    # [G] Specific conditioning mechanism (e.g., cross-attention, controlnet) is not detailed.
    # We assume the diffusion model is conditioned on the input image and uncertainty.
    pseudo_obs = diffusion_model.sample(rgb, depth, guidance=uncertainty)
    return pseudo_obs
```

### Formulas and Parameters
**Rendering Equation (NeRF):**
For a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, the predicted color is:
$$ \hat{\mathbf{C}} = \sum_{k=1}^K \hat{T}(t_k) \alpha(\sigma(t_k)\delta_k) \mathbf{c}(t_k) $$
where $\hat{T}(t_k) = \exp\left(-\sum_{k'=1}^{k-1} \sigma(t_{k'})\delta_{k'}\right)$, $\alpha(x) = 1 - \exp(-x)$, and $\delta_k = t_{k+1} - t_k$ [S1].

**Rendering Equation (3DGS):**
$$ \hat{\mathbf{C}} = \sum_{i=1}^n \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1}(1-\alpha_j) $$
where $\mathbf{c}_i$ is color from SH coefficients, and $\alpha_i$ is opacity from a 2D Gaussian multiplied by opacity logit $o$ [S1].

**Loss Function:**
The scene representation is optimized by minimizing the mean squared error (photometric loss) between rendered images and both real input images and pseudo-observations [S1].

**Parameters:**
*   **Densification Factor:** The method increases the number of training views by 5 to 10 times the original amount [S1].
*   **Alternation Iterations:** The process of generating pseudo-observations and training is repeated 5 times [S1].
*   **Backbone Settings:**
    *   Deceptive-NeRF: Nerfacto from NerfStudio with default proposal sampling, scene contraction, and appearance embeddings [S1].
    *   Deceptive-3DGS: Splatfacto from NerfStudio with default settings [S1].
*   **Initialization:** For 3DGS, Gaussians are randomly initialized without Colmap point clouds [S1]. A bounding box is defined by the outermost input cameras [S1].

### Edge Cases and Pitfalls
*   **Sparse Input Artifacts:** Standard NeRF/3DGS pipelines suffer from severe artifacts and overfitting with sparse inputs [S1]. The method mitigates this by densifying observations, but if the initial coarse rendering is too poor, the uncertainty guidance may fail to correct artifacts effectively [S1].
*   **Uncertainty Guidance Dependency:** Without the uncertainty measure, reconstruction quality drops significantly (e.g., PSNR 18.61 vs 20.44 on Hypersim), as the diffusion model cannot effectively remove artifacts in coarse renderings, leading to inconsistencies between pseudo-observations and real inputs [S1].
*   **Few-View Limit:** The method works effectively with as few as 3 input images, but performance degrades if the number of input views is extremely low (e.g., 1 or 2), though specific lower bounds are not explicitly stated beyond "as few as 3" [S1].

## Subtopics

### Deceptive-NeRF
Uses an MLP-based NeRF (Nerfacto) as the backbone. Suitable for scenes where volumetric representation is preferred or when integrating with existing NeRF pipelines. [Link](7371-simple-nerf.md)

### Deceptive-3DGS
Uses 3D Gaussian Splatting (Splatfacto) as the backbone. Suitable for real-time rendering requirements and scenes with complex geometry where Gaussian primitives excel. [Link](14428-sparse-view-and-few-shot-optimization.md)

## Operating characteristics
*   **Time Complexity:** Nearly tenfold faster than methods using diffusion models as per-step regularizers (e.g., SDS) because the diffusion model is inferred only during the pseudo-observation generation phase, not at every optimization step [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   On Hypersim dataset with 10 input views: PSNR 20.44, SSIM 0.748, LPIPS 0.173, Training Time 17min, Rendering FPS 5.3 [S1].
    *   Comparison with "Regularizer" approach on Hypersim: Regularizer achieved PSNR 19.31, SSIM 0.710, LPIPS 0.253, Training Time 2h40min, Rendering FPS 1.1 [S1].
    *   On DTU dataset scan 65 with 9 input views: Chamfer Distance 1.86 (vs NeRF 3.75, FreeNeRF 3.63, DiffusioNeRF 1.93) [S1].
*   **Scaling:** The method scales with the number of input views, showing consistent improvement as views increase from 3 to 10 on Hypersim [S1].
*   **Strengths:** High-quality reconstruction in sparse-view settings; fast training and rendering compared to diffusion-regularized methods; effective super-resolution capabilities [S1].
*   **Limitations:** Relies on the quality of the pre-trained diffusion model; requires careful tuning of the uncertainty measure; performance may degrade with extremely sparse inputs (<3 views) [S1].
*   **When to Use:** When high-quality novel view synthesis is required from a limited set of input images (3-10 views) and computational resources for per-step diffusion inference are limited [S1].
*   **When Not to Use:** When a large number of input views are available (standard NeRF/3DGS may suffice); when real-time training is not required and higher quality via regularizers is acceptable despite slower speed [S1].

## Background
Deceptive-NeRF/3DGS was proposed by X. Liu et al. in the paper "Deceptive-NeRF/3DGS: Diffusion-Generated Pseudo-Observations for High-Quality Sparse-View Reconstruction" [S1]. The method addresses the limitation of NeRF and 3DGS requiring large numbers of training views, which is often impractical for casual data collection using mobile devices [S1]. It leverages the prior knowledge embedded in large-scale 2D diffusion models to generate pseudo-observations, effectively "deceiving" the reconstruction algorithm into believing it has more data than it does [S1]. The approach was validated on Hypersim, mip-NeRF 360, and DTU datasets [S1].

## Sources
[S1] Deceptive-NeRF/3DGS: Diffusion-Generated Pseudo-Observations for High-Quality Sparse-View Reconstruction, https://arxiv.org/abs/2305.15171v4, fetched 2026-10-26
