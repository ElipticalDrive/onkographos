# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
Adaptive View Selection for Inpainting is a method for 3D scene de-occlusion that combines 2D appearance priors with multi-view consistency constraints to inpaint occluded regions in Neural Radiance Fields (NeRFs) [S1]. It addresses the limitation of single-view 2D inpainting, which often fails to maintain geometric and photometric consistency across different viewing angles [S1]. Unlike methods that uniformly apply 2D priors or rely solely on 3D consistency, this method dynamically selects between using 2D inpainted color priors and multi-view supplementary information based on the local texture richness (average gradient) of the occluded sub-regions [S1].

## How it works
The method operates on a pre-trained NeRF representation $F_\Theta$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $c$. It assumes access to a set of input views, a binary mask $M$ identifying the occluded region in at least one view, and a pre-trained 2D image inpainter (specifically LaMa [S1]).

### Inputs and Outputs
*   **Inputs**:
    *   Pre-trained NeRF parameters $\Theta$ (typically a coarse and fine MLP pair, denoted by outputs $\hat{C}_c$ and $\hat{C}_f$).
    *   Set of input views with ground truth colors $C(r)$ and depths $D(r)$.
    *   Binary mask $M$ indicating occluded pixels.
    *   Pre-trained 2D inpainter (LaMa) to generate color priors $C_l$.
*   **Outputs**:
    *   Updated NeRF parameters $\Theta'$ that render consistent colors for occluded regions.
    *   Refined segmentation masks $\hat{P}(r)$ for occluded areas across all views.

### Data Structures
*   **Ray Set $R$**: The set of all rays cast from the camera centers through the pixel grid.
*   **Sub-regions**: The occluded area in each view is divided into multiple sub-regions (patches) to evaluate local texture gradients individually [S1].
*   **Gradient Map**: For each sub-region, the average gradient magnitude is computed from the rendered image and the 2D inpainted image.

### Numbered Steps

1.  **2D Inpainting Prior Generation**:
    Apply the pre-trained 2D inpainter (LaMa) to the input views containing occlusions. This generates an inpainted image for each view. For a ray $r$ corresponding to a pixel in the occluded region, let $C_l(r)$ be the color value from the 2D inpainted image [S1].

2.  **Region Coding and Mask Refinement**:
    Train an object segmentation network (Region Coding Network) to predict a soft mask $\hat{P}(r)$ for the occluded area in each view. This network uses depth information and multi-view consistency to refine the initial user-provided or automatically detected masks $M$ [S1]. The network is optimized using a joint loss $L = \lambda_1 L_{region-encoding} + \lambda_2 L_{rgb} + \lambda_3 L_{depth}$, where $L_{depth}$ constrains the rendered depth $\hat{D}(r)$ to match ground truth $D(r)$ [S1].

3.  **Sub-region Division and Gradient Evaluation**:
    Divide the occluded area in each view into multiple sub-regions. For each sub-region, calculate the average gradient of the rendered result $\nabla \hat{g}$ and the average gradient of the 2D inpainting result $\nabla g$ [S1]. The gradient serves as an indicator of texture richness [S1].

4.  **Adaptive Weight Calculation**:
    Determine the adaptive weight $\lambda$ for the reconstruction loss based on the texture richness comparison. Let $\tau$ be a threshold parameter. The weight $\lambda$ is defined as:
    $$
    \lambda = \begin{cases} 
    1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
    0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g 
    \end{cases}
    $$
    If the rendered gradient is sufficiently rich compared to the 2D inpainted gradient, the method trusts the 3D model's ability to learn from other views ($\lambda=1$). Otherwise, it relies more heavily on the 2D prior ($\lambda=0$) [S1].

5.  **Loss Computation**:
    Compute the total reconstruction loss $L$ as a weighted sum of the non-occluded loss $L_{NonOcc}$ and the occluded loss $L_{Occ}$ [S1]:
    $$
    L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ}
    $$
    
    *   **Non-Occluded Loss ($L_{NonOcc}$)**:
        For rays $r \in R$ not in the occluded region (or weighted by $1-\hat{P}(r)$), the loss measures the difference between the predicted colors (coarse $\hat{C}_c$ and fine $\hat{C}_f$) and the ground truth $C(r)$ [S1]:
        $$
        L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ \|\hat{C}_c(r) - C(r)\|_2^2 + \|\hat{C}_f(r) - C(r)\|_2^2 \right]
        $$
    
    *   **Occluded Loss ($L_{Occ}$)**:
        For rays $r \in R$ in the occluded region (weighted by $\hat{P}(r)$), the loss measures the difference between the predicted colors and the 2D inpainted color prior $C_l(r)$ [S1]:
        $$
        L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ \|\hat{C}_c(r) - C_l(r)\|_2^2 + \|\hat{C}_f(r) - C_l(r)\|_2^2 \right]
        $$

6.  **Optimization**:
    Update the NeRF parameters $\Theta$ using an optimizer (e.g., Adam) to minimize $L$. The adaptive weight $\lambda$ is recalculated during training as the gradients evolve, allowing the model to switch between relying on 2D priors and multi-view consistency dynamically [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_view_selection_inpainting(nerf_model, input_views, masks_2d, lama_inpainter, tau):
    """
    Performs 3D inpainting with adaptive view selection.
    
    Args:
        nerf_model: Pre-trained NeRF with coarse and fine networks.
        input_views: List of input views (images, depths, poses).
        masks_2d: Binary masks for occluded regions in input views.
        lama_inpainter: Pre-trained 2D inpainter (LaMa).
        tau: Threshold for gradient comparison.
        
    Returns:
        Updated nerf_model parameters.
    """
    
    # Step 1: Generate 2D Inpainting Priors
    inpainted_views = []
    for view in input_views:
        # Apply LaMa to the occluded parts of the view
        inpainted_view = lama_inpainter.inpaint(view.image, masks_2d[view.id])
        inpainted_views.append(inpainted_view)
        
    # Step 2: Refine Masks using Region Coding Network
    # This step involves training a segmentation network to get soft masks P_hat
    # We assume this is done via a joint loss including depth and RGB consistency
    # For the purpose of the main inpainting loop, we use the refined masks
    refined_masks = train_region_coding_network(nerf_model, input_views, masks_2d)
    
    # Step 3 & 4 & 5: Optimization Loop
    optimizer = Adam(nerf_model.parameters(), lr=0.001)
    
    for epoch in range(num_epochs):
        total_loss = 0.0
        
        for view_id, view in enumerate(input_views):
            rays = cast_rays(view.pose, view.resolution)
            
            # Render current NeRF prediction
            colors_coarse, colors_fine, depths = nerf_model.render(rays)
            
            # Get 2D prior colors for occluded rays
            # C_l(r) is the color from the inpainted view at the corresponding pixel
            prior_colors = get_prior_colors(rays, inpainted_views[view_id])
            
            # Get refined mask values P_hat(r)
            mask_values = refined_masks[view_id].gather(rays.pixel_indices)
            
            # Divide occluded area into sub-regions for gradient evaluation
            # Note: In practice, this might be done per-patch or per-batch
            # Here we simplify to a global adaptive weight for the view, 
            # though the paper specifies sub-region level. 
            # To implement sub-region level, one would iterate over patches.
            
            # Calculate average gradients for the occluded region of this view
            # g_hat: gradient of rendered image in occluded region
            # g: gradient of inpainted image in occluded region
            grad_rendered = compute_average_gradient(colors_coarse, mask_values > 0.5)
            grad_prior = compute_average_gradient(prior_colors, mask_values > 0.5)
            
            # Determine adaptive weight lambda
            if grad_rendered > tau * grad_prior:
                lambda_weight = 1.0
            else:
                lambda_weight = 0.0
                
            # Calculate Non-Occluded Loss
            # L_NonOcc = sum((1 - P_hat) * (||C_c - C||^2 + ||C_f - C||^2))
            non_occluded_mask = 1.0 - mask_values
            loss_non_occ = torch.mean(non_occluded_mask * (
                torch.norm(colors_coarse - view.colors, p=2, dim=-1)**2 +
                torch.norm(colors_fine - view.colors, p=2, dim=-1)**2
            ))
            
            # Calculate Occluded Loss
            # L_Occ = sum(P_hat * (||C_c - C_l||^2 + ||C_f - C_l||^2))
            loss_occ = torch.mean(mask_values * (
                torch.norm(colors_coarse - prior_colors, p=2, dim=-1)**2 +
                torch.norm(colors_fine - prior_colors, p=2, dim=-1)**2
            ))
            
            # Total Loss for this view
            loss = lambda_weight * loss_non_occ + (1.0 - lambda_weight) * loss_occ
            
            total_loss += loss
            
        # Backpropagate and update
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        
    return nerf_model
```

### Parameters
*   $\tau$: Threshold for comparing rendered gradient to inpainted gradient. Not given in the sources.
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for the region coding network loss components. Not given in the sources.
*   $C_l(r)$: Color of the corresponding pixel in the 2D inpainting image for ray $r$. Used as an estimated color prior for occluded areas [S1].

### Edge Cases and Pitfalls
*   **Uniform Strategy Misjudgment**: Applying a single strategy to the entire occluded area can lead to errors if some parts have rich texture (benefiting from multi-view) and others are uniform (benefiting from 2D priors). The method mitigates this by dividing the occluded area into sub-regions and evaluating gradients individually [S1].
*   **2D Inpainter Inconsistency**: Single 2D image inpainting does not consider consistency in the real 3D scene. The method addresses this by selecting training views based on texture richness to ensure multi-view consistency [S1].
*   **Depth Accuracy**: The region coding network relies on depth information for mask refinement. Inaccurate depth estimates can lead to incorrect mask propagation across views [S1].

## Operating characteristics
*   **Time Complexity**: The method requires additional computation for 2D inpainting (LaMa) and the region coding network training. The main NeRF optimization loop is similar to standard NeRF but includes the calculation of adaptive weights and gradient comparisons per sub-region.
*   **Memory Complexity**: Requires storing the 2D inpainted images and the refined masks for all views.
*   **Strengths**: Effectively reduces the disturbing effect of 2D inpainters on 3D editing by leveraging multi-view consistency where appropriate. It adapts to the texture richness of the scene, providing better fidelity for detailed regions [S1].
*   **Limitations**: Relies on the quality of the pre-trained 2D inpainter (LaMa). Performance may degrade if the 2D inpainter produces artifacts that are not corrected by multi-view consistency. The method is evaluated on datasets like LLFF and SPIn-NeRF, which may not represent all real-world scenarios [S1].

## Background
The method was proposed in the paper "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1]. It addresses the challenge of removing obstacles from NeRFs and inpainting the resulting occluded areas. The key innovation is the training view selection mechanism based on regional average gradients, which allows the model to dynamically choose between 2D appearance priors and multi-view supplementary information [S1].

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
