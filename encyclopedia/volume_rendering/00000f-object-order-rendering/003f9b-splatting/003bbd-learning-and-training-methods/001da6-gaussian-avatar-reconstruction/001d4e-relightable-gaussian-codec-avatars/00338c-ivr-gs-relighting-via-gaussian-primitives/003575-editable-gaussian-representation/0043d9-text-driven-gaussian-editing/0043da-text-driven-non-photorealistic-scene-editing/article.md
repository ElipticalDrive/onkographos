# Text-Driven Non-Photorealistic Scene Editing

Also called: Image-driven NPSE, Text-Driven Gaussian Editing (for VolVis), Text-driven NPSE, Texture-Attribute Non-Photorealistic Scene Editing (NPSE)

## Overview
Text-Driven Non-Photorealistic Scene Editing (NPSE) stylizes volume visualization (VolVis) scenes represented by textured Gaussian primitives using text instructions, while Image-Driven NPSE performs the same task using a reference image prompt. Both variants optimize only the texture attributes of the Gaussian representation to preserve the underlying geometry and lighting, distinguishing them from methods that alter geometry or require iterative multi-view consistency updates. It sets itself apart from siblings like [StyleSplat](11108-text-driven-non-photorealistic-scene-editing.md) by embedding texture maps within Gaussian primitives to express complex style patterns and reduce elliptical artifacts, leveraging a one-time update strategy for text-driven editing to avoid the blurriness associated with iterative approaches in semi-transparent scenes.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained TexGS-VolVis scene (Gaussian primitives with geometry $\mu$, shading $\beta$, and texture attributes), a text prompt (for text-driven) or an image prompt $I_{prmt}$ (for image-driven), and optionally a 2D segmentation mask for region-specific editing.
*   **Outputs:** An optimized TexGS-VolVis scene with modified texture attributes that reflect the stylized appearance.

**Data Structures:**
The method operates on Gaussian primitives enhanced with a texture attribute. Each primitive stores:
*   Geometry attributes (e.g., mean position $\mu$).
*   Shading attributes (e.g., $\beta$).
*   Texture attributes: A texture map (set of texels) associated with the primitive.
*   Opacity: A scalar value controlling transparency.
*   View-independent color $c_{ind}$: Used only if the texture attribute is disabled.

**Numbered Steps:**

1.  **Initialization:**
    *   For **Image-Driven NPSE**: Randomly initialize all Gaussian texture maps. Define the view-independent color using the texture-sampled color, ignoring palette colors.
    *   For **Text-Driven NPSE**: Initialize the scene opacity at $2\times$ its target value.
    *   Fix all geometry attributes ($\mu$) and shading attributes ($\beta$) to preserve consistent geometry and lighting.

2.  **Prompt Generation (Text-Driven Only):**
    *   Use the IP2P model (based on Stable Diffusion) to generate edited images from unedited multi-view images and the text instruction.
    *   Set classifier-free guidance scale for input multi-view images to 1.25.
    *   Set classifier-free guidance scale for text prompts to 12.5.
    *   [G] Ensure the IP2P model is loaded and configured with the specified guidance scales before inference.

3.  **Optimization Loop:**
    *   Select training views. For text-driven NPSE, use only six views (front, back, left, right, top, bottom) to reduce ambiguity. For image-driven NPSE, use the full set of training views (e.g., 162 uniformly sampled via icosphere sampling).
    *   Render the scene $R$ from the selected training views using Blinn-Phong shading.
    *   Calculate the loss function based on the editing type:
        *   **Image-Driven:** Use $L_{img}$ (defined below).
        *   **Text-Driven:** Use an RGB reconstruction loss between the rendered image $R$ and the IP2P-generated edited image.
    *   Update only the texture attributes (or $c_{ind}$ if texture is disabled) via gradient descent.
    *   **Opacity Adjustment (Text-Driven Only):** Gradually reduce the scene's opacity from the initial $2\times$ target value to $0.5\times$ over fixed iteration intervals to prevent occlusion of unedited primitives and reduce blurriness from inconsistent edits.

4.  **Iteration Count:**
    *   For **Image-Driven NPSE**: Optimize for 3,000 iterations.
    *   For **Text-Driven NPSE**: Optimize for 1,500 iterations using the IP2P editing images.

5.  **Region-Specific Editing (Optional):**
    *   If editing specific regions, use a 2D-lift-3D segmentation approach:
        *   Render a view from TexGS-VolVis.
        *   Use a 2D segmentation model (e.g., SAM) with user-specified point prompts to generate a 2D mask.
        *   Lift the 2D mask to 3D by identifying and segmenting the corresponding Gaussian primitives.
        *   Apply the editing optimization only to the primitives in the segmented group.

**Formulas:**

The loss function for image-driven NPSE is:
$$ L_{img} = \lambda_s L_{NNFM}(R, I_{prmt}) + (1 - \lambda_s) \| C(R) - C(I_{prmt}) \|_2 $$

Where:
*   $R$ is the rendered image from TexGS-VolVis.
*   $I_{prmt}$ is the image prompt.
*   $C$ is a pretrained CLIP model.
*   $L_{NNFM}$ is the Neural Network Feature Matching (NNFM) loss using a pretrained VGG-16 network to capture fine stylization details.
*   $\| C(R) - C(I_{prmt}) \|_2$ is the L2 distance between feature vectors extracted by CLIP, encouraging global style consistency.
*   $\lambda_s$ is the hyperparameter balancing fine details and global style.

If the texture attribute is not used, the appearance is edited by updating the view-independent color $c_{ind}$ within each primitive.

**Parameters:**
*   $\lambda_s$: Hyperparameter balancing fine stylization details and global style consistency.
    *   $\lambda_s = 0.0$: Emphasizes global style patterns.
    *   $\lambda_s = 0.5$: Intermediate balance.
    *   $\lambda_s = 0.9$: Emphasizes finer, more localized style details (default).
    *   $\lambda_s = 1.0$: Maximum emphasis on fine details.
*   Training Image Resolution:
    *   800×800 for image-driven editing (balances quality and fitting time).
    *   512×512 for text-driven editing (avoids out-of-memory issues with IP2P).
*   Training Views: 162 uniformly sampled via icosphere sampling is sufficient; increasing beyond this yields marginal gains.
*   Iterations: 3,000 for image-driven; 1,500 for text-driven.

**Edge Cases and Pitfalls:**
*   **Text Prompt Accuracy:** Text-based editing relies heavily on the IP2P model. Inaccurate or vague text prompts lead to erroneous editing results [S1].
*   **Opacity Issues:** High opacity causes occlusion, leaving uncovered primitives unedited. Low opacity leads to blurry stylization due to inconsistent IP2P edits across views. The gradual opacity reduction strategy mitigates this [S1].
*   **Storage:** The texture attribute can occupy significant storage space, especially with high texel counts, resulting in larger model sizes compared to other Gaussian Splatting methods [S1].
*   **Segmentation Efficiency:** The 2D-lift-3D segmentation method may be inefficient for scenes requiring multiple rounds of segmentation, as it cannot immediately generate a 3D segmentation from a single 2D mask [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def text_driven_npse(texgs_volvis, text_prompt, num_iterations=1500):
    """
    Performs text-driven non-photorealistic scene editing.
    """
    # 1. Generate edited images using IP2P
    # Use 6 training views: front, back, left, right, top, bottom
    training_views = get_6_directional_views()
    edited_images = []
    for view in training_views:
        unedited_image = render(texgs_volvis, view)
        # IP2P model parameters: guidance_scale_image=1.25, guidance_scale_text=12.5
        edited_img = ip2p_model(unedited_image, text_prompt, 
                                guidance_scale_image=1.25, 
                                guidance_scale_text=12.5)
        edited_images.append(edited_img)

    # 2. Initialize opacity
    target_opacity = texgs_volvis.get_target_opacity()
    current_opacity = 2.0 * target_opacity
    texgs_volvis.set_opacity(current_opacity)

    # 3. Fix geometry and shading
    texgs_volvis.freeze_geometry()
    texgs_volvis.freeze_shading()

    # 4. Optimization loop
    for i in range(num_iterations):
        # Adjust opacity gradually to 0.5x target
        progress = i / num_iterations
        current_opacity = target_opacity * (2.0 - 1.5 * progress)
        texgs_volvis.set_opacity(current_opacity)

        # Render and compute loss
        total_loss = 0
        for view, edited_img in zip(training_views, edited_images):
            rendered_img = render(texgs_volvis, view)
            # RGB reconstruction loss
            loss = mse_loss(rendered_img, edited_img)
            total_loss += loss

        # Update only texture attributes
        gradients = compute_gradients(total_loss, texgs_volvis.texture_attributes)
        update_parameters(texgs_volvis.texture_attributes, gradients)

    return texgs_volvis

def image_driven_npse(texgs_volvis, image_prompt, lambda_s=0.9, num_iterations=3000):
    """
    Performs image-driven non-photorealistic scene editing.
    """
    # 1. Initialize texture maps
    texgs_volvis.initialize_texture_maps_randomly()
    # Define view-independent color from texture-sampled color
    texgs_volvis.set_view_independent_color_from_texture()

    # 2. Fix geometry and shading
    texgs_volvis.freeze_geometry()
    texgs_volvis.freeze_shading()

    # 3. Optimization loop
    for i in range(num_iterations):
        # Select a random training view
        view = random_sample_training_view()
        rendered_img = render(texgs_volvis, view)

        # Compute L_img
        nnfm_loss = nnfm_loss_function(rendered_img, image_prompt) # Uses VGG-16
        clip_loss = l2_distance(clip_model(rendered_img), clip_model(image_prompt))
        loss = lambda_s * nnfm_loss + (1 - lambda_s) * clip_loss

        # Update texture attributes
        gradients = compute_gradients(loss, texgs_volvis.texture_attributes)
        update_parameters(texgs_volvis.texture_attributes, gradients)

    return texgs_volvis
```

## Subtopics

### [Image-Driven Non-Photorealistic Scene Editing (NPSE)](11113-image-driven-non-photorealistic-scene-editing-npse.md)
Stylizes volume visualization scenes by transferring the artistic style of a reference image onto a 3D Gaussian representation, ensuring multi-view consistency. It distinguishes itself from sibling methods like [StyleSplat](11108-text-driven-non-photorealistic-scene-editing.md) by embedding texture attributes within Gaussian primitives, allowing each primitive to express multiple colors and reducing elliptical artifacts, while leveraging a combined NNFM and CLIP loss strategy to capture both local and global style patterns.

## Operating characteristics
*   **Time Complexity:** The method requires iterative optimization. Image-driven NPSE takes approximately 3,000 iterations, while text-driven NPSE takes 1,500 iterations. Fitting time is influenced by the number of primitives and training image resolution. For example, on the supernova dataset, fitting time increases from 4.24 minutes (400×400 resolution) to 6.98 minutes (800×800 resolution) [S1].
*   **Memory Complexity:** The texture attribute can occupy significant storage space, especially as the number of texels increases, resulting in a larger model size than other Gaussian Splatting methods [S1]. Text-driven editing uses lower resolution (512×512) to avoid out-of-memory issues with the IP2P model [S1].
*   **Measured Performance:** On the supernova dataset at 800×800 resolution, the method achieves a training PSNR of 29.16 dB and SSIM of 0.9515, using 53,356 primitives [S1].
*   **Strengths:** Real-time rendering, expressive style patterns via texture attributes, geometry-consistent editing without iterative updates for text-driven NPSE, and flexible partial editing via 2D-lift-3D segmentation.
*   **Limitations:** High storage requirements for texture attributes, reliance on IP2P model quality for text-driven editing, and potential inefficiency in 2D-lift-3D segmentation for complex scenes.
*   **When to Use:** When high-quality, expressive non-photorealistic stylization of volume visualization scenes is required, and real-time rendering is needed.
*   **When Not to Use:** When storage space is limited, or when precise text prompts are not available for text-driven editing.

## Background
TexGS-VolVis is the first 2DGS-based framework in VolVis aimed at expressive scene editing. It builds upon existing NeRF and 3DGS scene editing approaches [6, 7, 9, 22] guided by text instructions. The method was introduced in the paper "TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting" [S1]. It addresses the challenges of applying iterative editing strategies to complex and semi-transparent VolVis scenes, which often result in blurry appearances or geometry-inconsistent edits.

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
