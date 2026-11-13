# RePaint-NeRF


## Overview
RePaint-NeRF is a framework for editing 3D content in Neural Radiance Fields (NeRF) using text prompts by leveraging a pre-trained diffusion model to guide changes in designated 3D content, ensuring view consistency and scene integrity [S1]. It solves the problem of modifying specific objects within a complex 3D scene without altering the surrounding environment or introducing artifacts like "black holes" where previously occluded geometry is exposed [S1]. It sets itself apart from standard NeRF optimization by decoupling the process into a semantic mask extraction stage and a diffusion-guided editing stage, using CLIP features for segmentation and Stable Diffusion for generation [S1].

## How it works
RePaint-NeRF operates in two distinct stages: Semantic Mask Extraction and Text-to-3D Content Editing. It builds on [NeRF Training Loop](970-nerf-training-loop.md) by adding a semantic feature branch and diffusion-based loss terms.

### Inputs and Outputs
**Inputs:**
*   A set of training images $I$ with known camera poses $P$ (extracted via COLMAP) [S1].
*   A target text prompt describing the new object (e.g., "a blue rose") [S1].
*   A background text prompt (BGT) describing the surrounding environment (e.g., "leaves") to fill exposed areas [S1].
*   A user-selected patch on a training image to define the target object for mask generation [S1].

**Outputs:**
*   An edited NeRF model $\theta$ that renders the scene with the modified object while preserving the background [S1].

### Data Structures
*   **NeRF Model ($g_\theta$):** Implemented using Instant-NGP [Müller et al., 2022], utilizing a multi-resolution hash grid structure [S1]. It outputs color $C$ and density $\sigma$ for 3D points [S1].
*   **Semantic Feature Network ($s$):** A network outputting a feature vector $f$ for each 3D point $x$. The feature map size is $H \times W \times 512$ [S1].
*   **Depth Map:** Size $H \times W \times 1$, normalized to range $[0, 1]$, predicted by MiDAS [Ranftl et al., 2022] [S1].
*   **Feature Map ($F_I$):** Rendered semantic feature map from the feature network, size $H \times W \times 512$ [S1].
*   **Mask ($I_{mask}$):** Binary mask of size $H \times W \times 1$ [S1].

### Stage 1: Semantic Mask Extraction
The goal is to extract a view-consistent semantic mask for the target object.

1.  **Initialize:** Train the NeRF color/density modules and the semantic feature module $s: x \to f$ jointly.
2.  **Loss Function:** Minimize $L_{first-stage}$:
    $$ L_{first-stage} = L_{color} + \lambda_{feature} L_{feature} + \lambda_{depth} L_{depth} $$
    where:
    *   $L_{color}$ is the standard photometric loss between rendered and ground truth colors [S1].
    *   $L_{feature}$ encourages the rendered feature map $F_I$ to match the features from a pre-trained semantic model (LSeg) [S1].
    *   $L_{depth}$ is a coarse depth supervision loss:
        $$ L_{depth} = \sum_{r \in R} (D(r) - \hat{D}(r))^2 $$
        where $D(r)$ is the depth predicted by MiDAS and $\hat{D}(r)$ is the depth predicted by the NeRF [S1].
3.  **Optimization:** Use Adam optimizer with learning rate $1e-2$ and batch size 4096 for 2000 steps [S1].
4.  **Mask Generation:**
    *   Render the feature map $F_I$ for training views.
    *   Compute the mean feature $F_{patch}$ of the user-selected patch.
    *   Compute the mask:
        $$ I_{mask}^{H \times W \times 1} = 1(Sim(F_{patch}, F_I) > \alpha) $$
        where $Sim$ is a similarity function (e.g., cosine similarity) and $\alpha$ is a threshold [S1].
    *   *Note:* Patch features are preferred over text features for mask generation to allow higher thresholds and more accurate segmentation [S1].

### Stage 2: Text-to-3D Content Editing
The goal is to modify the masked region according to the text prompt while preserving the unmasked region and filling "black holes" (previously occluded areas).

1.  **Initialize:** Use the NeRF from Stage 1. Pre-train the base NeRF for 3000 steps using standard NeRF loss [S1].
2.  **Loss Function:** Minimize $L_{repaint}$:
    $$ L_{repaint} = \lambda_{unmask} L_{unmask} + \lambda_{clip} L_{clip} $$
    where:
    *   $L_{unmask}$ constrains the unmasked region to remain stable:
        $$ L_{unmask} = \sum_{r \in R} (C(r)_{unmask} - \hat{C}(r)_{unmask})^2 $$
        where:
        $$ C(r)_{unmask} = C(r) \times 1(I_{mask} < 0.5) $$
        $$ \hat{C}(r)_{unmask} = \hat{C}(r) \times 1(I_{mask} < 0.5) $$
        This ensures that pixels not in the target mask do not change significantly [S1].
    *   $L_{clip}$ guides the generation of background content in exposed areas:
        $$ L_{clip} = -Sim(Z_I, Z_{BGT}) $$
        where $Z_I$ is the latent feature of the rendered image $I$ encoded by CLIP, and $Z_{BGT}$ is the latent feature of the background text prompt encoded by CLIP [S1].
3.  **Diffusion Guidance:** Use Stable Diffusion [Rombach et al., 2022] to guide the update direction of the NeRF. The gradient $\nabla_\theta L_{SDS}$ is used to update the NeRF parameters $\theta$ [S1].
4.  **Optimization:** Use Adan optimizer with learning rate decaying from $1e-3$ to $1e-4$ and batch size 1. Train for 10,000 steps (generation/editing phase) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def repaint_nerf(training_images, poses, target_prompt, background_prompt, patch_selection):
    """
    RePaint-NeRF editing pipeline.
    """
    # Stage 1: Semantic Mask Extraction
    # Initialize NeRF with Instant-NGP structure and semantic feature branch
    nerf_model = InstantNGPNeRF()
    feature_network = SemanticFeatureNetwork()
    
    # Optimization parameters for Stage 1
    optimizer_stage1 = Adam(lr=1e-2, batch_size=4096)
    
    for step in range(2000):
        # Sample rays
        rays = sample_rays(training_images, poses)
        
        # Forward pass
        colors, depths, features = nerf_model(rays), feature_network(rays)
        gt_colors, gt_depths = get_ground_truth(rays)
        gt_features = get_semantic_features(rays) # From LSeg/CLIP
        
        # Compute losses
        l_color = mse_loss(colors, gt_colors)
        l_feature = mse_loss(features, gt_features)
        l_depth = mse_loss(depths, gt_depths) # Supervised by MiDAS
        
        l_first_stage = l_color + lambda_feature * l_feature + lambda_depth * l_depth
        
        # Backward pass
        optimizer_stage1.zero_grad()
        l_first_stage.backward()
        optimizer_stage1.step()

    # Generate Mask
    # Render feature map for all training views
    feature_maps = render_feature_maps(nerf_model, feature_network, training_images, poses)
    
    # Compute mean feature of selected patch
    f_patch = mean_feature(patch_selection, feature_maps)
    
    # Compute binary mask
    # Sim is cosine similarity, alpha is threshold
    masks = (cosine_similarity(f_patch, feature_maps) > alpha).float()

    # Stage 2: Text-to-3D Content Editing
    # Pre-train base NeRF for 3000 steps (standard NeRF loss)
    train_base_nerf(nerf_model, training_images, poses, steps=3000)
    
    # Optimization parameters for Stage 2
    optimizer_stage2 = Adan(lr=1e-3, lr_decay=1e-4, batch_size=1)
    
    # Load Stable Diffusion and CLIP models
    diffusion_model = StableDiffusion()
    clip_encoder = CLIP()
    
    # Encode background prompt
    z_bg = clip_encoder.encode(background_prompt)
    
    for step in range(10000):
        # Sample ray
        ray = sample_ray(training_images, poses)
        
        # Forward pass through NeRF
        color, depth, _ = nerf_model(ray)
        
        # Render full image for diffusion guidance
        rendered_image = render_image(nerf_model, poses)
        
        # Compute Unmasked Loss
        # Apply mask to ensure unmasked regions stay stable
        mask = masks[ray.view_id]
        color_unmasked = color * (mask < 0.5)
        gt_color_unmasked = get_gt_color(ray) * (mask < 0.5)
        l_unmask = mse_loss(color_unmasked, gt_color_unmasked)
        
        # Compute CLIP Loss for background filling
        # Encode rendered image and background prompt
        z_img = clip_encoder.encode(rendered_image)
        l_clip = -cosine_similarity(z_img, z_bg)
        
        # Compute Diffusion Loss (SDS)
        # This guides the masked region to match target_prompt
        # Note: SDS gradient is computed w.r.t the rendered image, then backprop to NeRF
        l_sds = compute_sds_loss(diffusion_model, rendered_image, target_prompt)
        
        # Total Loss
        # Note: The paper defines L_repaint = lambda_unmask * L_unmask + lambda_clip * L_clip
        # However, it also states SDS guides the update direction.
        # Combining them:
        l_repaint = lambda_unmask * l_unmask + lambda_clip * l_clip + l_sds
        
        # Backward pass
        optimizer_stage2.zero_grad()
        l_repaint.backward()
        optimizer_stage2.step()

    return nerf_model
```

### Parameters
*   $\lambda_{feature}$: Weight for feature loss in Stage 1. Value not explicitly given in text [S1].
*   $\lambda_{depth}$: Weight for depth loss in Stage 1. Value not explicitly given in text [S1].
*   $\lambda_{unmask}$: Weight for unmasked region stability in Stage 2. Typical value: 100. Too large values severely restrict the diffusion model's generative ability and slow generation speed [S1].
*   $\lambda_{clip}$: Weight for CLIP loss in Stage 2. Used to balance change of target object and other regions. Value not explicitly given in text [S1].
*   $\alpha$: Threshold for mask generation. Value not explicitly given in text [S1].
*   Learning Rate (Stage 1): $1e-2$ [S1].
*   Learning Rate (Stage 2): $1e-3$ decaying to $1e-4$ [S1].
*   Batch Size (Stage 1): 4096 [S1].
*   Batch Size (Stage 2): 1 [S1].
*   Steps (Stage 1): 2000 [S1].
*   Steps (Stage 2 Pre-training): 3000 [S1].
*   Steps (Stage 2 Generation): 10000 [S1].

### Edge Cases and Pitfalls
*   **Black Holes:** If the newly generated object is smaller than the original, previously covered areas become "black holes" because the content is unknown to the model [S1]. Mitigated by adding a background prompt (BGT) and CLIP loss [S1].
*   **Depth Noise:** Noise in depth estimation causes imprecise semantic mask segmentation. Mitigated by adding coarse depth supervision from MiDAS [S1].
*   **Unwanted Background Changes:** If $\lambda_{unmask} = 0$, the diffusion model may alter the background completely. Gradually increasing $\lambda_{unmask}$ enforces shape/appearance preservation [S1].
*   **Computation Time:** The process is time-consuming and space-consuming. Actual generation time depends on the complexity of the edit (e.g., 5 minutes for color change, >40 minutes for shape change) [S1].

## Subtopics

### Unmasked Region Weighting
A scalar weight $\lambda_{unmask}$ applied to the loss function that constrains the unmasked region to prevent unwanted diffusion model changes. When $\lambda_{unmask} = 0$, no constraints are applied to the background, allowing the diffusion model to alter it completely. Gradually increasing $\lambda_{unmask}$ enforces shape/appearance preservation. Typical value is 100. Too large values severely restrict the diffusion model's generative ability and slow generation speed [S1].

### Mask+SDS
A variant of the editing guidance where the Stable Diffusion Score Distillation Sampling (SDS) loss is used to guide the masked part. It produces significantly better texture and shape than Mask+CLIP because SDS provides stronger gradient guidance for the diffusion model [S1].

## Operating characteristics
*   **Time Complexity:** High. The method requires two stages of optimization. Stage 1 takes 2000 steps. Stage 2 takes 3000 steps for pre-training and 10,000 steps for generation. Each iteration renders a whole view, consuming significant memory and time [S1].
*   **Memory Complexity:** High. Requires storing the Instant-NGP hash grids, feature maps, and depth maps. Batch size of 1 in Stage 2 is used to manage GPU memory [S1].
*   **Hardware:** Tested on a single NVIDIA RTX 3090 GPU [S1].
*   **Resolution:** LLFF resized to 504 x 378; Blender resized to 400 x 400 [S1].
*   **Strengths:** View-consistent editing; ability to edit complex scenes; handles "black holes" via background prompts and CLIP loss [S1].
*   **Limitations:** Time-consuming; space-consuming; limited by view angle (occluded content may not be edited correctly if not visible in training views) [S1].

## Background
RePaint-NeRF was proposed in 2023 by Kobayashi et al. (IJCAI-23) [S1]. It builds on DreamFusion [Poole et al., 2022] for text-to-3D generation and DFF [Kobayashi et al., 2022] for semantic feature extraction. It uses Instant-NGP [Müller et al., 2022] for acceleration, Stable Diffusion [Rombach et al., 2022] for guidance, and CLIP [Radford et al., 2021] for semantic understanding [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
