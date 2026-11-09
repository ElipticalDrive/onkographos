# GO-NeRF
Also called: Generating Objects in Neural Radiance Fields

## Overview
GO-NeRF is a pipeline for generating text-prompt-controlled 3D virtual objects at user-specified locations within an existing 3D environment represented as a NeRF, while preserving unchanged scene content [S1]. It distinguishes itself from general scene editing methods by employing a compositional rendering formulation that separates the pre-trained scene from the generated object, using optimized 3D-aware opacity maps to handle occlusions and ensure compatibility with various NeRF representations such as InstantNGP [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained NeRF representing the existing 3D scene; a text prompt describing the object to generate; three 2D points selected by the user on a rendered image of the scene to define the region of interest (ROI) [S1].
*   **Outputs:** A final composited image $I_v$ containing the original scene with the new object seamlessly integrated [S1].

**Data Structures:**
*   **Scene NeRF:** The pre-trained representation of the environment.
*   **Object NeRF:** A new NeRF representation optimized specifically for the generated object within the defined 3D bounding box.
*   **3D Bounding Box:** A volumetric region defined in 3D space, derived from the user's 2D selection [S1].

**Steps:**

1.  **Region Definition:** The user selects three points on a rendered image of the pre-trained scene. These 2D points are converted into a 3D bounding box within the scene using depth information, perspective projection, and cross-product operations [S1].
2.  **Scene Rendering:** The pre-trained scene NeRF is rendered using pre-defined cameras to generate RGB-D images $(S, D)$ of the 3D scene [S1].
3.  **Object Generation (Optimization):** A new NeRF representation is optimized within the 3D bounding box. This optimization produces RGB images $G_v$ and opacity maps $O_v$ through volume rendering, guided by text prompts [S1].
4.  **Compositional Rendering:** The rendered scene $S$ and the generated object $G_v$ are composited using the 3D-aware opacity map $O_v$ to handle occlusions and create the final output $I_v$ [S1].

**Formulas and Optimization:**

The object generation step optimizes the Object NeRF using Score Distillation Sampling (SDS) to distill 2D text-guided image inpainting priors from diffusion models [S1]. The RGB image $G_v$ and opacity map $O_v$ are produced via volume rendering [S1].

To address oversaturation common in SDS-based results, a saturation regularizer is introduced to align the saturation of synthesized objects with the overall tone of the scene [S1].

To eliminate artifacts that share color with the scene background, the scene context is randomly replaced with a random background at a certain ratio during training [S1].

A reference-image-guided feature space loss is applied to guide the style of the generated objects using feature space similarity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GO-NeRF Optimization Loop
# Inputs: pre-trained_scene_nerf, text_prompt, user_2d_points, reference_image (optional)
# Outputs: optimized_object_nerf

def go_nerf_pipeline(pretrained_scene_nerf, text_prompt, user_2d_points, reference_image=None):
    # Step 1 & 2: Region Definition
    # Convert 2D points to 3D bounding box using depth, perspective projection, and cross-product
    bounding_box = define_3d_box_from_2d_points(user_2d_points, pretrained_scene_nerf)
    
    # Step 3: Scene Rendering
    # Render pre-trained scene to get RGB-D images
    # Cameras are pre-defined
    scene_rgb, scene_depth = render_scene(pretrained_scene_nerf)
    
    # Initialize Object NeRF within the bounding box
    object_nerf = initialize_nerf(bounding_box)
    
    # Optimization Loop
    for iteration in range(num_iterations):
        # Sample rays within the bounding box for the object
        rays = sample_rays_in_box(bounding_box)
        
        # Render object RGB and Opacity
        # G_v and O_v are produced through volume rendering
        object_rgb, object_opacity = render_object(object_nerf, rays)
        
        # Randomly replace scene context with random background to mitigate artifacts
        # This is done at a certain ratio during training
        if random() < background_replacement_ratio:
            scene_context = random_background()
        else:
            scene_context = scene_rgb
            
        # Composite Scene and Object
        # I_v is created by blending scene and generated content using optimized opacity maps
        final_image = composite_images(scene_context, object_rgb, object_opacity)
        
        # Calculate Losses
        
        # 1. SDS Loss: Distill 2D text-guided image inpainting priors
        # Uses diffusion model to guide optimization based on text_prompt
        sds_loss = compute_sds_loss(final_image, text_prompt)
        
        # 2. Saturation Regularizer: Align object saturation with scene tone
        # Addresses SDS-based oversaturation
        saturation_loss = compute_saturation_regularization(object_rgb, scene_rgb)
        
        # 3. Feature Space Loss: Guide style using reference image
        # Only applied if reference_image is provided
        style_loss = 0
        if reference_image is not None:
            style_loss = compute_feature_space_similarity(final_image, reference_image)
            
        total_loss = sds_loss + saturation_loss + style_loss
        
        # Update Object NeRF parameters
        update_nerf(object_nerf, total_loss)
        
    return object_nerf

def define_3d_box_from_2d_points(points_2d, scene_nerf):
    # Use depth information, perspective projection, and cross-product operations
    # to convert 2D points to 3D box
    pass

def render_scene(scene_nerf):
    # Render using pre-defined cameras
    pass

def render_object(object_nerf, rays):
    # Volume rendering to produce RGB and Opacity
    pass

def composite_images(scene, object_rgb, object_opacity):
    # Blend using 3D-aware opacity map
    pass

def compute_sds_loss(image, prompt):
    # Score Distillation Sampling from diffusion model
    pass

def compute_saturation_regularization(obj_rgb, scene_rgb):
    # Align saturation of object with scene tone
    pass

def compute_feature_space_similarity(image1, image2):
    # Feature space similarity
    pass
```

**Parameters:**
*   **Background Replacement Ratio:** The ratio at which the scene context is randomly replaced with a random background during training. Not given in the sources.
*   **Number of Iterations:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Oversaturation:** SDS-based results can suffer from oversaturation issues [S1].
*   **Artifacts:** Artifacts may arise during optimization if they share color with the scene background [S1].

## Operating characteristics
Not given in the sources.

## Background
GO-NeRF was proposed to address the complex process of generating high-quality 3D objects and seamlessly integrating them into existing 3D scenes for virtual reality content creation [S1]. It leverages scene context to generate 3D objects within the scene, employing compositional rendering to avoid unintended modifications to the original scene [S1]. The method was demonstrated on public datasets including forward-facing and 360-degree scenes [S1].

## Sources
[S1] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
