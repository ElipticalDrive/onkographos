# Depth-Guided Sequential Editing

Also called: ViCA-NeRF, ViCA-NeRF Sequential Editing, View-Consistency-Aware 3D Editing of Neural Radiance Fields, View-Consistency-Aware NeRF

## Overview
Depth-Guided Sequential Editing (ViCA-NeRF) is a two-stage method for editing Neural Radiance Fields (NeRFs) using text instructions, which ensures multi-view consistency by sequentially editing key 2D views and propagating those edits to the rest of the scene via geometric projection and learned regularization [S1]. It differs from [Instruct-NeRF2NeRF](12265-instruct-nerf2nerf.md) by explicitly enforcing consistency through depth-guided pixel projection and latent code alignment, avoiding the need to integrate a diffusion model into the NeRF training loop, resulting in editing speeds approximately 3 times faster [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained NeRF model (specifically `nerfacto` from NeRFStudio [S1]), a set of input views (images, camera poses, and coarse depth maps rendered from the source NeRF), and a text editing instruction.
*   **Outputs:** A refined NeRF model representing the edited 3D scene, and a corresponding set of edited 2D images.

**Data Structures:**
*   **View Data:** Each view $i$ consists of an RGB image $I_i$, a coarse depth map $D_i$, a camera intrinsic matrix $K_i$, and a camera pose $\xi_i$ [S1].
*   **Key View Sequence:** An ordered list of view indices selected for sequential editing.
*   **Modified Mask:** A binary mask per view tracking which pixels have already been overwritten by previous projections to prevent overwriting [S1].

**Parameters:**
*   $\phi$: Maximum desired modified ratio for key view selection, set to $\phi = 0.3$ [S1].
*   $lr_{thresh}$: Reprojection error threshold for valid correspondences, set to $5$ pixels [S1].
*   $T_{warmup}$: Warm-up iterations for the diffusion model; $10$ for face-centric scenes, $30$ for outdoor scenes [S1].
*   $T_{fine}$: Fine-tuning iterations for the NeRF; $10,000$ for general scenes, $\approx 5,000$ for small scenes [S1].
*   $T_{train}$: Initial training iterations for the source NeRF; $30,000$ [S1].

**Steps:**

1.  **Source NeRF Training:** Train a pre-trained NeRF (e.g., `nerfacto`) on the original unedited data for $T_{train} = 30,000$ iterations [S1].
2.  **Depth Extraction:** Render coarse depth maps $D_i$ for all input views $i$ from the trained source NeRF [S1].
3.  **Key View Selection:**
    *   Select a random view as the first key view.
    *   For subsequent selections, calculate the modified ratio $\rho_i$ for each unselected view $i$, defined as the fraction of pixels already modified by previous projections.
    *   Compute a weight $w_i$ for each unselected view:
        $$
        w_i = \begin{cases} 
        \rho_i & \text{if } \rho_i < \phi \\
        \phi - (\rho_i - \phi) & \text{if } \rho_i \ge \phi 
        \end{cases}
        $$
    *   Select the view with the largest weight $w_i$ as the next key view [S1].
    *   Repeat until all views' minimum $\rho$ exceeds a certain threshold (implied: all views are sufficiently covered or no valid candidates remain) [S1].
4.  **Sequential Editing and Projection:**
    *   Iterate through the selected key views in order.
    *   **Edit Key View:** Apply Instruct-Pix2Pix to the current key view $j$ using the text prompt to generate an edited image $I'_j$. A warm-up procedure is applied: run the diffusion process for $T_{warmup}$ iterations to shift the main distribution of 2D edits [S1].
    *   **Geometric Projection:** Project the edited pixels from $I'_j$ to all other views $i$ (including non-key views and subsequent key views) using the depth maps and camera poses.
        *   Calculate 3D points for view $i$: $p_i = P^{-1}(D_i, K_i, \xi_i)$ [S1].
        *   Reproject $p_i$ to view $j$ to find corresponding pixel coordinates.
        *   Calculate reprojection error $lr$. If $lr < lr_{thresh}$ ($5$px), the correspondence is valid.
        *   Copy the pixel value from $I'_j$ to the corresponding location in view $i$, but **only** if that pixel has not yet been modified (checked via the Modified Mask) [S1]. This creates a "mixup" dataset where edits are mixed but not overwritten.
5.  **Blending Refinement (Learned Regularization):**
    *   The projected mixup images may contain noise due to depth inaccuracies.
    *   Run the diffusion process (Instruct-Pix2Pix) multiple times on each mixup image.
    *   Average the resulting latent codes from these runs.
    *   Decode the averaged latent code into a single final image. This corrects noisy depth correspondences and stabilizes the edit, accepting slight image smoothness [S1].
6.  **NeRF Fine-Tuning:**
    *   Train the source NeRF on the fully refined, view-consistent edited dataset for $T_{fine}$ iterations ($10,000$ for general, $5,000$ for small scenes) [S1].
7.  **Optional Post-Refinement:**
    *   Re-render images from the fine-tuned NeRF.
    *   Apply the blending refinement model (Step 5) again to further improve consistency.
    *   *Note:* Skip this step for large-scale outdoor scenes (e.g., 'campsite', 'farm') as high complexity prevents further consistency improvement [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_guided_sequential_editing(source_neurf, views, text_prompt):
    """
    source_neurf: Pre-trained NeRF model (nerfacto)
    views: List of view objects containing {image, pose, intrinsics}
    text_prompt: String instruction for editing
    """
    
    # Step 1: Train source NeRF
    # source_neurf.train(views, iterations=30000) 
    # [G] Assuming standard NeRF training loop is handled by the framework.
    
    # Step 2: Extract Depth
    depth_maps = {}
    for view in views:
        depth_maps[view.id] = source_neurf.render_depth(view)
        
    # Step 3: Select Key Views
    key_views = []
    modified_masks = {v.id: np.zeros_like(v.image) for v in views} # Binary mask, 1 if modified
    unselected_views = [v.id for v in views]
    
    # Start with a random view
    start_view_id = random.choice(unselected_views)
    key_views.append(start_view_id)
    unselected_views.remove(start_view_id)
    
    phi = 0.3
    
    while unselected_views:
        weights = {}
        for vid in unselected_views:
            # Calculate modified ratio rho
            rho = np.sum(modified_masks[vid]) / modified_masks[vid].size
            
            # Calculate weight w
            if rho < phi:
                weights[vid] = rho
            else:
                weights[vid] = phi - (rho - phi)
                
        # Select view with max weight
        next_view_id = max(weights, key=weights.get)
        key_views.append(next_view_id)
        unselected_views.remove(next_view_id)
        
        # Check termination condition (all views sufficiently covered)
        # [G] The paper states "ends when all views' minimum rho exceeds a certain threshold".
        # We approximate this by checking if the minimum rho in the scene is high enough,
        # or simply iterating until all views are processed if the logic implies full coverage.
        # For implementation safety, we break if no valid candidates or max iterations.
        if min(weights.values()) < 0.01: # Placeholder threshold for "exceeds certain threshold"
             # Actually, the logic selects based on max weight. 
             # The termination is likely when all views have been selected or rho is too high for selection.
             # Given the loop structure, we just proceed until unselected_views is empty or logic dictates stop.
             pass 

    # Step 4 & 5: Sequential Editing, Projection, and Blending
    edited_images = {v.id: v.image.copy() for v in views}
    
    for key_vid in key_views:
        # Edit the key view
        # Warm-up: Run diffusion for T_warmup iterations to shift distribution
        # [G] Instruct-Pix2Pix usually takes image + prompt. 
        # We assume a function `edit_image` that handles the diffusion steps.
        edited_key_img = edit_image_with_warmup(edited_images[key_vid], text_prompt, warmup_iters=30 if is_outdoor else 10)
        
        # Project to all other views
        for target_vid in [v.id for v in views if v.id != key_vid]:
            # Get depth and pose for target
            D_target = depth_maps[target_vid]
            K_target = views_by_id[target_vid].intrinsics
            xi_target = views_by_id[target_vid].pose
            
            # Get depth and pose for source (key view)
            D_source = depth_maps[key_vid]
            K_source = views_by_id[key_vid].intrinsics
            xi_source = views_by_id[key_vid].pose
            
            # Project 3D points from target to source to find correspondence
            # Calculate 3D points for target view
            # [G] P_inv is the inverse projection matrix function using D, K, xi
            points_3d = unproject_depth(D_target, K_target, xi_target)
            
            # Reproject these 3D points to the source view (key_vid)
            # [G] project_points function returns 2D coordinates and reprojection error
            u_src, v_src, lr = project_points(points_3d, K_source, xi_source)
            
            # Filter valid correspondences
            valid_mask = lr < 5.0 # lr_thresh
            
            # Apply modification only if pixel not yet modified
            for idx in range(len(u_src)):
                if valid_mask[idx]:
                    u, v = int(u_src[idx]), int(v_src[idx])
                    # Check bounds
                    if 0 <= u < edited_images[target_vid].shape[1] and 0 <= v < edited_images[target_vid].shape[0]:
                        # Check if already modified
                        if modified_masks[target_vid][v, u] == 0:
                            # Copy pixel from edited key view to target view
                            # Note: edited_key_img is the result of editing the key view
                            # We need to map the pixel from the *edited* key view image
                            # But wait, the projection is from Target 3D -> Source 2D.
                            # So we take the pixel from the Edited Key View at (u, v)
                            # and put it into the Target View at the corresponding location?
                            # No, the target view's pixel location is determined by the target's own grid.
                            # The correspondence tells us which pixel in the SOURCE view corresponds to the pixel in the TARGET view.
                            # So, for each pixel (y, x) in TARGET view:
                            #   1. Project to 3D.
                            #   2. Project to SOURCE view -> (u_src, v_src).
                            #   3. If valid, take color from SOURCE edited image at (u_src, v_src).
                            #   4. Write to TARGET image at (y, x).
                            pass 
                            
            # Re-implementing the loop correctly for vectorization or explicit iteration
            # For each pixel (y, x) in target view:
            for y in range(D_target.shape[0]):
                for x in range(D_target.shape[1]):
                    if modified_masks[target_vid][y, x] == 0:
                        # Get 3D point for (y,x) in target
                        p = unproject_depth_pixel(D_target, K_target, xi_target, y, x)
                        # Project p to source view
                        u_s, v_s, lr_val = project_point_to_view(p, K_source, xi_source)
                        
                        if lr_val < 5.0:
                            # Check bounds in source view
                            if 0 <= u_s < edited_images[key_vid].shape[1] and 0 <= v_s < edited_images[key_vid].shape[0]:
                                # Copy color from edited key view
                                edited_images[target_vid][y, x] = edited_images[key_vid][v_s, u_s]
                                modified_masks[target_vid][y, x] = 1

    # Step 5: Blending Refinement
    refined_images = {}
    for vid, img in edited_images.items():
        # Run diffusion multiple times, average latents, decode
        refined_images[vid] = blend_refine(img, text_prompt)
        
    # Step 6: NeRF Fine-Tuning
    # Create dataset from refined_images
    edited_dataset = create_dataset(refined_images, views)
    source_neurf.fine_tune(edited_dataset, iterations=10000) # 5000 for small scenes
    
    # Step 7: Optional Post-Refinement
    # if not is_large_outdoor_scene:
    #     re_rendered = source_neurf.render_all()
    #     final_refined = blend_refine_all(re_rendered, text_prompt)
    #     source_neurf.fine_tune_again(final_refined)
        
    return source_neurf

def edit_image_with_warmup(image, prompt, warmup_iters):
    """
    Simulates Instruct-Pix2Pix with warm-up.
    """
    # [G] Instruct-Pix2Pix is a latent diffusion model.
    # Warm-up implies running the denoising steps with the prompt to shift the latent space.
    # Then we perform the actual edit.
    # For implementation, this would call the specific IP2Pix API.
    # Here we assume a black-box function that returns the edited image.
    return ip2pix_edit(image, prompt, warmup_steps=warmup_iters)

def blend_refine(image, prompt):
    """
    Runs diffusion multiple times, averages latents, decodes.
    """
    # [G] This is the learned regularization step.
    # 1. Encode image to latent z.
    # 2. Run DDIM/DPMSolver steps for N times with prompt.
    # 3. Average the resulting latents z_avg.
    # 4. Decode z_avg to image.
    return ip2pix_blend(image, prompt, num_runs=5) # [G] num_runs not specified, assume >1
```

## Operating characteristics
*   **Time Complexity:** The method is significantly faster than joint training approaches. It achieves approximately 3 times faster editing speed compared to [Instruct-NeRF2NeRF](12265-instruct-nerf2nerf.md) [S1].
*   **Memory Complexity:** Requires storing depth maps for all views and the modified masks. The blending refinement step requires multiple passes through the diffusion model's latent space, increasing VRAM usage during the refinement stage [S1].
*   **Performance:** Evaluated on scenes ranging from faces to outdoor environments. It provides higher levels of consistency and detail compared to the state of art [S1].
*   **Strengths:** Explicitly enforces multi-view consistency via geometric projection and latent code alignment. Offers improved controllability (by editing key views) and efficiency (less GPU time) [S1].
*   **Limitations:**
    *   Inherits limitations from Instruct-Pix2Pix: inability to perform large spatial changes (e.g., adding/removing large objects) and difficulty with highly detailed instructions involving interactions between parts [S1].
    *   Blurry results due to high-frequency information loss (texture on face, beard, hair) when the mixup image passes through the diffusion model, which attempts to reduce noise introduced during projection [S1].
    *   Post-refinement is ineffective for large-scale outdoor scenes due to high complexity [S1].
*   **When to use:** Suitable for 3D scene editing and AR applications where multi-view consistency is critical and computational efficiency is desired. Best for scenes where the user can define key views that capture the desired edit [S1].
*   **When not to use:** Avoid for large-scale outdoor scenes if post-refinement is expected to improve results. Avoid if large spatial manipulations (adding/removing objects) are required [S1].

## Background
*   **Origin:** ViCA-NeRF was introduced as the first view-consistency-aware method for 3D editing with text instructions [S1].
*   **History:** It builds upon the limitations of [Instruct-NeRF2NeRF](12265-instruct-nerf2nerf.md), which suffers from convergence problems and slower speeds due to iterative diffusion model updates during NeRF training [S1]. ViCA-NeRF decouples the editing from the training loop by using geometric and learned regularization in a pre-processing stage [S1].
*   **Usage:** Evaluated on various scenes and text prompts, ranging from faces to outdoor scenes. Code is available at https://dongjiahua.github.io/VICA-NeRF [S1].

## Sources
[S1] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
