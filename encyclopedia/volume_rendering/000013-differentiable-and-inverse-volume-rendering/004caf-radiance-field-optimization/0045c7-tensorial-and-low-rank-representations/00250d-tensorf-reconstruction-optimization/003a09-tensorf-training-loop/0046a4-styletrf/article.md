# StyleTRF

Also called: StyleTRF (S3)

## Overview
StyleTRF is a method for stylized view generation that decouples style adaptation from view capture by fine-tuning the appearance component of a pre-optimized Tensorial Radiance Field (TensoRF) using sparse stylized priors [S1]. It sets itself apart from joint optimization methods like SNeRF by freezing the geometric density parameters of the scene representation, thereby preserving crisp geometry while adapting appearance, and by using a deterministic CNN-based stylization module (Johnson et al.) to ensure 3D consistency across viewpoints [S1].

## How it works
**Inputs:**
- A set of posed ground-truth images of a scene.
- A reference style image.
- Camera poses (obtained via COLMAP for real scenes or Blender for synthetic scenes) [S1].

**Outputs:**
- A fine-tuned TensoRF representation capable of rendering stylized novel views.

**Data Structures:**
The scene is represented using the VM-48 variant of TensoRF, which employs Vector-Matrix decomposition (a special case of Block Term Decomposition) [S1].
- **Density Tensor ($T_\sigma$):** Stores volumetric density. These parameters are frozen during the style adaptation phase [S1].
- **Appearance Tensor ($T_C$):** Stores color/appearance vectors. These parameters are optimized during the style adaptation phase [S1].
- **Voxel Grid:** The effective resolution starts at $128^3$ and is upsampled to $640^3$ for real-world scenes (or $300^3$ for synthetic scenes) [S1].
- **Interpolation:** Bi-linear interpolation is used for matrices, and linear interpolation is used for vectors within the VM-decomposed representation during ray queries and upsampling [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def style_trf(ground_truth_images, camera_poses, style_image):
    # Phase 1: Pre-optimization of TensoRF
    # Initialize TensoRF VM-48 representation
    tensorf = initialize_tensorf_vm48(effective_resolution=128**3)
    
    # Train TensoRF on ground truth to learn geometry and appearance
    # Optimizer: Adam, LR = 0.02
    # Loss: L1 norm + Total Variation (TV) loss
    for iteration in range(15000):
        # Sample 4096 rays
        rays = sample_rays(camera_poses, count=4096)
        
        # Render colors using volumetric rendering
        rendered_colors = volumetric_render(tensorf, rays)
        
        # Calculate losses
        l1_loss = calculate_l1_loss(rendered_colors, ground_truth_images[rays])
        tv_loss = calculate_tv_loss(tensorf)
        total_loss = l1_loss + tv_loss
        
        # Update TensoRF parameters (density and appearance)
        tensorf.update_parameters(total_loss)
        
        # Upsampling schedule
        if iteration >= 2000 and iteration % 1000 == 0 and iteration <= 5000:
            tensorf.upsample()
            # Re-initialize Adam optimizer LR to 0.02 after upsampling [S1]
            
    # Phase 2: Preprocessing Stylization Module
    # Train Johnson et al. CNN for the specific style on COCO2014
    # Takes ~20 minutes [S1]
    stylization_cnn = train_johnson_style_cnn(style_image, dataset="COCO2014")
    
    # Phase 3: Generate and Stylize Priors
    # Render sparse novel views (20-30 views in spiral trajectory)
    novel_view_cameras = generate_spiral_trajectory(count=30)
    novel_views = render_novel_views(tensorf, novel_view_cameras)
    
    # Stylize these views using the pre-trained CNN
    stylized_priors = [stylization_cnn(view) for view in novel_views]
    
    # Phase 4: Fine-tuning Appearance
    # Freeze density parameters of TensoRF
    tensorf.freeze_density()
    
    # Optimize only appearance vectors using stylized priors
    # Optimizer: Adam, LR = 0.02
    # Iterations: 1000
    for iteration in range(1000):
        # Sample rays towards the same camera positions as priors
        rays = sample_rays(novel_view_cameras, count=len(stylized_priors))
        
        # Render current appearance
        rendered_colors = volumetric_render(tensorf, rays)
        
        # Calculate loss against stylized priors
        style_loss = calculate_l1_loss(rendered_colors, stylized_priors)
        
        # Update only appearance parameters
        tensorf.update_appearance_parameters(style_loss)
        
    return tensorf

def volumetric_render(tensorf, rays):
    """
    Computes color C from transmittance tau_q, color c_q, density sigma, and step size Delta.
    C = sum_{q=1}^{Q} tau_q * c_q
    tau_q = product_{p=1}^{q-1} exp(-sigma_p * Delta_p)
    """
    color = 0
    transmittance = 1.0
    for q in range(1, Q):
        # Sample density sigma_q and color c_q from TensoRF at ray point q
        sigma_q = tensorf.query_density(rays[q])
        c_q = tensorf.query_color(rays[q])
        delta_q = rays.step_size[q]
        
        color += transmittance * c_q
        transmittance *= exp(-sigma_q * delta_q)
    return color

def calculate_tv_loss(tensorf):
    """
    L_TV = (1/N) * sum( sqrt(Delta^2 T_sigma) + 0.1 * sqrt(Delta^2 T_C) )
    Delta^2 is squared difference between neighboring values.
    N is total parameters.
    Density and appearance weighted 10:1 (density weight 1, appearance weight 0.1 in formula context relative to sum, 
    but text says weighted in ratio 10:1 respectively, implying density is 10x more important or coefficient is 1 vs 0.1).
    """
    # Implementation detail: sum over all neighboring pairs in T_sigma and T_C
    # N = total number of parameters in TensoRF representation
    pass
```

**Formulas:**
- **Volumetric Rendering:**
  $$C = \sum_{q=1}^{Q} \tau_q c_q$$
  where $\tau_q = \prod_{p=1}^{q-1} \exp(-\sigma_p \Delta_p)$. $C$ is the final color, $\tau_q$ is the transmittance, $c_q$ is the color, $\sigma$ is the density, and $\Delta$ is the step size [S1].

- **Total Variation Loss:**
  $$L_{TV} = \frac{1}{N} \sum \left( \sqrt{\Delta^2 T_\sigma} + 0.1 \sqrt{\Delta^2 T_C} \right)$$
  where $\Delta^2$ is the squared difference between neighboring values in the tensors, $N$ is the total number of parameters in the TensoRF representation $T$, $T_\sigma$ represents the density value, and $T_C$ represents the appearance value. The density and appearance are weighted in a ratio of 10:1 [S1].

**Parameters:**
- **TensoRF Variant:** VM-48 (Vector-Matrix decomposition) [S1].
- **Optimizer:** Adam [S1].
- **Learning Rate:** 0.02 (re-initialized to 0.02 after upsampling) [S1].
- **Voxel Grid Resolution:** Effective resolution starts at $128^3$. Upsampled every 1000 iterations starting at iteration 2000 until 5000 iterations, reaching $640^3$ for real-world scenes and $300^3$ for synthetic scenes [S1].
- **Training Iterations (Pre-optimization):** 15,000 iterations [S1].
- **Training Iterations (Style Fine-tuning):** 1,000 iterations [S1].
- **Rays per Iteration:** 4,096 rays [S1].
- **Number of Stylized Priors:** 20–30 (or 30–40) randomly sampled camera positions around the object [S1].
- **Stylization Module:** Johnson et al. [20] (trained on COCO2014) [S1].

**Edge Cases and Pitfalls:**
- **Optimizing from scratch (Strategy S1):** Fails because stylized priors may not share exact geometry with ground truth, leading to geometric artifacts [S1].
- **Unconstrained optimization (Strategy S2):** Optimizing both density and appearance simultaneously leads to fuzzy geometry and artifacts in thin structures because limit-free optimization alters geometry components [S1].
- **Stylization Module Choice:** Using Gatys et al. [10] causes inconsistencies across nearby viewpoints because it starts with white noise and depends on initialization/optimizer, making it unreliable for 3D consistency [S1].
- **Micro-details:** Strategy S2 struggles to capture micro-details and may lose geometry [S1].

## Operating characteristics
- **Time Complexity:**
  - Pre-optimization of TensoRF: 10–15 minutes [S1].
  - Stylization Module Training (per style): ~20 minutes [S1].
  - Style Fine-tuning: 40–50 seconds [S1].
  - Rendering (800x800 image): 4–5 seconds [S1].
- **Memory Complexity:**
  - Memory footprint of TensoRF VM-48: 10–15 MB [S1].
- **Strengths:**
  - Decouples style adaptation from view capture.
  - Preserves crisp geometry by freezing density.
  - Fast style adaptation (40–50 seconds).
  - Low memory footprint compared to MLP-based NeRFs (~1GB) [S1].
  - Stable stylization across close-by viewpoints using Johnson et al. [S1].
- **Limitations:**
  - Requires pre-optimization of the scene.
  - Relies on the quality of the sparse priors; insufficient priors may lead to incomplete style transfer.
- **When to use:**
  - When crisp geometry preservation is critical and fast style adaptation is desired.
  - When 3D consistency across viewpoints is required.
- **When not to use:**
  - When joint optimization of geometry and style is preferred (e.g., if the style implies geometric changes).
  - When real-time training is required (pre-optimization takes 10-15 mins).

## Background
StyleTRF was introduced in the paper "StyleTRF: Stylizing Tensorial Radiance Fields" presented at ICVGIP’22 in December 2022 [S1]. It builds upon the TensoRF representation [6] and the Johnson et al. stylization method [20]. It addresses limitations in previous methods like SNeRF [29], which uses Gatys et al. [10] and joint optimization, leading to higher training times and potential 3D inconsistencies [S1].

## Sources
[S1] StyleTRF: Stylizing Tensorial Radiance Fields✱, https://doi.org/10.1145/3571600.3571643, fetched 2026-10-26
