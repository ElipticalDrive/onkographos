# GO-NeRF

Also called: Generating Objects in Neural Radiance Fields

## Overview
GO-NeRF is a pipeline for generating text-prompt-controlled 3D virtual objects at user-specified locations within an existing 3D environment represented as a NeRF, while preserving unchanged scene content [S1]. It differs from standard NeRF training by decoupling the object generation into a localized 3D bounding box and using compositional rendering to integrate the new object with the pre-trained scene, ensuring compatibility with various NeRF representations [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained NeRF representing the existing 3D scene; a text prompt describing the object to generate; three 2D points selected by the user on a rendered image of the scene to define the object's location.
*   **Outputs:** A final composited image $I_v$ containing the original scene with the newly generated object seamlessly integrated [S1].

**Data Structures:**
*   **Scene NeRF:** A pre-trained neural radiance field mapping $(x, y, z) \to (\sigma, c)$ [S1].
*   **Object NeRF:** A new neural radiance field optimized within a specific 3D bounding box, mapping $(x, y, z) \to (\sigma_{obj}, c_{obj})$ [S1].
*   **3D Bounding Box:** Defined by three 2D points projected into 3D space [S1].
*   **RGB-D Images:** Rendered from the scene NeRF, consisting of RGB color $S$ and depth $D$ [S1].
*   **Opacity Maps:** $O_v$, representing the 3D-aware opacity of the generated object [S1].

**Steps:**

1.  **Region Definition:** The user selects three points on a rendered image of the pre-trained scene. These 2D points are converted into a 3D bounding box within the scene using depth information, perspective projection, and cross-product operations [S1].
2.  **Scene Rendering:** Render the pre-trained scene NeRF using pre-defined cameras $C$ to generate RGB-D images $(S, D)$ of the 3D scene [S1].
3.  **Object Generation (Optimization):** Optimize a new NeRF representation within the 3D bounding box. This step uses Score Distillation Sampling (SDS) to distill 2D text-guided image inpainting priors from diffusion models, guided by the text prompt [S1]. The optimization produces RGB images $G_v$ and opacity maps $O_v$ through volume rendering [S1].
4.  **Compositional Rendering:** Composite the rendered scene and the generated object using the 3D-aware opacity map to handle occlusions and create the final output $I_v$ [S1].

**Formulas:**

The object's RGB image $G_v$ and opacity map $O_v$ are produced via volume rendering within the bounding box:
$$ G_v = \sum_{i=1}^{N} T_i \alpha_i c_i $$
$$ O_v = \sum_{i=1}^{N} T_i \alpha_i $$
where $T_i = \prod_{j=1}^{i-1} e^{-\sigma_j \delta_j}$ is the accumulated transmittance, $\sigma_i$ is the density of the object NeRF, $c_i$ is the color, and $\delta_i$ is the interval length [S1].

The final composited image $I_v$ is created by blending the scene $S$ and the generated object $G_v$ using the opacity map $O_v$:
$$ I_v = G_v \cdot O_v + S \cdot (1 - O_v) $$
This formulation ensures that the original scene content outside the desired editing area is preserved and occlusions are managed correctly [S1].

**Optimization Strategy and Loss Functions:**

The optimization of the Object NeRF is driven by several components:

1.  **Score Distillation Sampling (SDS):** Distills 2D text-guided image inpainting priors from diffusion models to guide the generation based on the text prompt [S1].
2.  **Saturation Regularizer:** Addresses SDS-based oversaturation by aligning the saturation of synthesized objects with the overall tone of the scene [S1].
3.  **Random Background Replacement:** To eliminate artifacts that share color with the scene background, the scene context is randomly replaced with a random background at a certain ratio during training [S1].
4.  **Reference-Image-Guided Feature Space Loss:** Uses feature space similarity to guide the style of the generated objects based on a reference image [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def go_nerf_pipeline(scene_nerf, text_prompt, user_points_2d, reference_image=None):
    # Step 1: Region Definition
    # Convert 2D points to 3D bounding box using depth and perspective projection
    depth_map = render_depth(scene_nerf)
    bounding_box = project_2d_to_3d(user_points_2d, depth_map)
    
    # Step 2: Scene Rendering
    # Render RGB-D images of the scene
    cameras = get_predefined_cameras()
    scene_rgb, scene_depth = render_scene(scene_nerf, cameras)
    
    # Initialize Object NeRF within the bounding box
    object_nerf = initialize_nerf(bounding_box)
    
    # Step 3: Object Generation (Optimization Loop)
    for step in range(num_training_steps):
        # Sample rays within the bounding box
        rays = sample_rays_in_box(bounding_box, cameras)
        
        # Render object RGB and opacity
        object_rgb, object_opacity = render_object(object_nerf, rays)
        
        # Compute SDS loss from diffusion model using text prompt
        # This distills 2D inpainting priors
        sds_loss = compute_sds_loss(object_rgb, text_prompt, diffusion_model)
        
        # Apply saturation regularizer
        scene_saturation = compute_saturation(scene_rgb)
        object_saturation = compute_saturation(object_rgb)
        saturation_loss = compute_saturation_alignment_loss(object_saturation, scene_saturation)
        
        # Apply random background replacement to mitigate artifacts
        if random() < background_replacement_ratio:
            random_bg = generate_random_background()
            # Modify loss computation to use random background context
            # This helps prevent objects from blending incorrectly with scene colors
            context_loss = compute_context_loss(object_rgb, random_bg)
        else:
            context_loss = compute_context_loss(object_rgb, scene_rgb)
            
        # Apply reference-image-guided feature space loss if reference provided
        if reference_image is not None:
            ref_features = extract_features(reference_image)
            obj_features = extract_features(object_rgb)
            feature_loss = compute_feature_similarity_loss(ref_features, obj_features)
        else:
            feature_loss = 0.0
            
        # Total loss
        total_loss = sds_loss + saturation_loss + context_loss + feature_loss
        
        # Update Object NeRF parameters
        update_parameters(object_nerf, total_loss)
        
    # Step 4: Compositional Rendering
    # Final render of object
    final_object_rgb, final_object_opacity = render_object(object_nerf, cameras)
    
    # Composite scene and object
    final_image = final_object_rgb * final_object_opacity + scene_rgb * (1 - final_object_opacity)
    
    return final_image
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of training steps for the Object NeRF and the cost of SDS evaluation from the diffusion model. SDS is computationally expensive as it requires multiple forward passes through the diffusion model per gradient step [S1].
*   **Memory Complexity:** Requires storing the pre-trained Scene NeRF and the newly optimized Object NeRF. The Object NeRF is localized, so its memory footprint is smaller than a full-scene NeRF [S1].
*   **Strengths:** Preserves unchanged scene content; handles occlusions via 3D-aware opacity maps; compatible with various NeRF representations (e.g., InstantNGP, standard NeRF) [S1].
*   **Limitations:** Susceptible to oversaturation without the saturation regularizer; artifacts may occur if the object color matches the scene background, mitigated by random background replacement [S1].
*   **When to use:** When creating virtual reality content by adding specific, text-described objects to an existing 3D scene at precise locations [S1].

## Background
GO-NeRF was introduced to address the gap in generating high-quality 3D objects within existing 3D environments for virtual reality content creation. It leverages recent advances in text-to-image diffusion models and NeRF representations [S1]. The method was published in IEEE Transactions on Visualization and Computer Graphics in 2025 [S1].

## Sources
[S1] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
