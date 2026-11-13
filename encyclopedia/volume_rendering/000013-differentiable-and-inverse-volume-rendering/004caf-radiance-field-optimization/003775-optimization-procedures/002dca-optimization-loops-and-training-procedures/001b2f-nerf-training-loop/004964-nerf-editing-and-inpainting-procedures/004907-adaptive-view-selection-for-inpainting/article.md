# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
This method addresses the inconsistency of single-view 2D inpainting in 3D scenes by introducing a training view selection mechanism that leverages multi-view correlation [S1]. It distinguishes itself from standard inpainting by dynamically choosing between relying on a pretrained 2D inpainter's appearance prior or supplementary information from other views based on the texture richness (average gradient) of the occluded regions [S1].

## How it works
**Inputs:**
*   A pretrained NeRF model $f_\theta$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
*   A set of input views with known camera poses.
*   A binary mask $M$ indicating occluded regions in the target view (and inferred masks for other views via an object segmentation network [S1]).
*   A pretrained 2D image inpainter (specifically LaMa [S1]) to generate color priors $C_l$ for occluded pixels.
*   Hyperparameter $\tau$ (threshold for gradient comparison) [S1].

**Outputs:**
*   Optimized NeRF parameters $\theta$ that render consistent colors for occluded areas, utilizing either multi-view geometry or 2D priors as appropriate.

**Data Structures:**
*   **Ray Set $R$**: The set of rays cast from the target view through the occluded region.
*   **Sub-regions**: The occluded area is divided into multiple sub-regions (patches) to evaluate local texture richness [S1].
*   **Predicted Mask $\hat{P}(r)$**: A soft or hard mask value for ray $r$, indicating occlusion status (1 if occluded, 0 if visible) [S1].
*   **Color Predictions**: $\hat{C}_c(r)$ and $\hat{C}_f(r)$ denote the predicted colors from the coarse and fine NeRF networks, respectively [S1].

**Numbered Steps:**

1.  **Initial 2D Inpainting**: Use the pretrained 2D inpainter (LaMa) to generate an initial color prior $C_l(r)$ for all pixels in the occluded region of the target view [S1].
2.  **Region Division**: Divide the occluded area in the target view into multiple sub-regions (patches) [S1].
3.  **Gradient Evaluation**: For each sub-region, calculate the average gradient of the rendered result $\nabla \hat{g}$ and the average gradient of the 2D inpainting result $\nabla g$ [S1].
4.  **Adaptive Weight Calculation**: Determine the adaptive weight $\lambda$ for the reconstruction loss based on the gradient comparison [S1]:
    $$
    \lambda = \begin{cases} 
    1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
    0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g 
    \end{cases}
    $$
    Here, $\nabla \hat{g}$ represents the texture richness of the current NeRF rendering, and $\nabla g$ represents the texture richness of the 2D inpainting prior [S1]. If the NeRF rendering has sufficient texture (gradient), $\lambda=1$ prioritizes the NeRF's multi-view consistency. If the NeRF rendering is smooth/featureless, $\lambda=0$ prioritizes the 2D inpainter's prior [S1].
5.  **Loss Computation**: Compute the total reconstruction loss $L$ as a weighted sum of non-occluded loss $L_{NonOcc}$ and occluded loss $L_{Occ}$ [S1]:
    $$
    L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ}
    $$
    Where:
    *   $L_{NonOcc}$ constrains the modeling of visible areas using ground truth colors $C(r)$ [S1]:
        $$
        L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ \|\hat{C}_c(r) - C(r)\|_2^2 + \|\hat{C}_f(r) - C(r)\|_2^2 \right]
        $$
    *   $L_{Occ}$ constrains the modeling of occluded areas using the 2D inpainting prior $C_l(r)$ [S1]:
        $$
        L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ \|\hat{C}_c(r) - C_l(r)\|_2^2 + \|\hat{C}_f(r) - C_l(r)\|_2^2 \right]
        $$
    *   Note: The summation is over rays $r$ in the set $R$ corresponding to the occluded region [S1].
6.  **Optimization**: Update the NeRF parameters $\theta$ to minimize $L$. The method implicitly relies on the object segmentation network to provide accurate masks $\hat{P}(r)$ for both the target and other views, ensuring multi-view consistency during training [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_adaptive_inpainting_loss(
    rays_in_occluded_region: List[Ray],
    nerf_coarse: NeuralRadianceField,
    nerf_fine: NeuralRadianceField,
    mask: Tensor,  # Shape: (H, W), 1 for occluded, 0 for visible
    gt_colors: Tensor,  # Shape: (H, W, 3), ground truth colors
    inpaint_prior_colors: Tensor,  # Shape: (H, W, 3), from LaMa
    tau: float,
    sub_region_size: int
) -> Tensor:
    """
    Computes the adaptive reconstruction loss for inpainting occluded areas.
    
    Args:
        rays_in_occluded_region: List of Ray objects corresponding to occluded pixels.
        nerf_coarse: Coarse NeRF model.
        nerf_fine: Fine NeRF model.
        mask: Binary mask of the target view (1 if occluded).
        gt_colors: Ground truth colors for the target view.
        inpaint_prior_colors: Colors from 2D inpainter (LaMa) for occluded areas.
        tau: Threshold for gradient comparison.
        sub_region_size: Size of patches for gradient evaluation.
        
    Returns:
        Total loss scalar.
    """
    # 1. Render coarse and fine colors for rays in the occluded region
    # Note: In practice, rays are batched. Here we assume a loop for clarity of logic.
    coarse_colors = []
    fine_colors = []
    
    for ray in rays_in_occluded_region:
        c_c, _ = nerf_coarse.sample(ray)
        c_f, _ = nerf_fine.sample(ray)
        coarse_colors.append(c_c)
        fine_colors.append(c_f)
        
    coarse_colors = torch.stack(coarse_colors)
    fine_colors = torch.stack(fine_colors)
    
    # 2. Get ground truth and prior colors for these rays
    # Map rays back to pixel coordinates to index into masks and color tensors
    pixel_indices = [ray.pixel_coords for ray in rays_in_occluded_region]
    gt_vals = gt_colors[pixel_indices]
    prior_vals = inpaint_prior_colors[pixel_indices]
    
    # 3. Calculate average gradients for sub-regions
    # The mask and colors are 2D images. We need to evaluate gradients per sub-region.
    # This requires iterating over sub-regions of the occluded mask.
    
    # Initialize total loss components
    L_non_occ_sum = 0.0
    L_occ_sum = 0.0
    
    # We need to handle non-occluded rays as well for the full loss L_NonOcc
    # But the adaptive weight lambda is specific to occluded regions/sub-regions.
    # The source defines L = lambda * L_NonOcc + (1-lambda) * L_Occ.
    # This implies lambda is a scalar for the whole loss or per-region?
    # Source says: "lambda is the adaptive weight... if average gradient of rendered result 
    # in a sub-region exceeds...". This suggests lambda might be computed per sub-region,
    # but the final loss formula L = lambda L_NonOcc + (1-lambda) L_Occ uses a single lambda.
    # Interpretation: The method likely computes a global lambda based on the average 
    # gradient of the entire occluded area, or applies the loss formulation per sub-region.
    # Given the formula structure L = lambda L_NonOcc + (1-lambda) L_Occ, 
    # and the condition on "a sub-region", it is most likely that the loss is computed 
    # per sub-region, or lambda is a global average. 
    # However, standard implementation of such adaptive weighting usually applies the 
    # weight to the specific term it governs. 
    # Let's assume the loss is computed globally, and lambda is determined by the 
    # overall texture richness of the occluded area, or the formula applies to the 
    # occluded part specifically. 
    # Re-reading: "L = lambda L_NonOcc + (1-lambda) L_Occ". 
    # L_NonOcc is over all rays (or non-occluded rays). L_Occ is over occluded rays.
    # The condition "if average gradient ... in a sub-region" suggests local adaptation.
    # To be implementable from text alone, we will compute a global lambda based on 
    # the average gradient of the entire occluded region, as a single scalar lambda 
    # is used in the final equation.
    
    # Calculate average gradient of rendered result in occluded region
    # Gradient magnitude of the rendered image (coarse or fine, usually coarse for speed)
    rendered_img_coarse = render_image_from_rays(rays_in_occluded_region, nerf_coarse, mask_shape=mask.shape)
    grad_rendered = compute_average_gradient(rendered_img_coarse, mask) # Only over occluded pixels
    
    # Calculate average gradient of 2D inpainting result in occluded region
    inpaint_img = inpaint_prior_colors.clone() # Assuming inpaint_prior_colors is the full image or we fill non-occluded with GT
    # For non-occluded, we don't care about inpaint prior. 
    # We only compare gradients in occluded areas.
    grad_inpaint = compute_average_gradient(inpaint_img, mask)
    
    # Determine lambda
    if grad_rendered > tau * grad_inpaint:
        lambda_val = 1.0
    else:
        lambda_val = 0.0
        
    # Compute L_NonOcc
    # Rays NOT in occluded region
    non_occ_rays = get_non_occluded_rays(rays_in_occluded_region, mask) # Helper to get all rays minus occluded
    if non_occ_rays:
        c_c_non = render_rays(non_occ_rays, nerf_coarse)
        c_f_non = render_rays(non_occ_rays, nerf_fine)
        gt_non = get_gt_colors(non_occ_rays)
        
        loss_c_non = torch.mean((c_c_non - gt_non)**2)
        loss_f_non = torch.mean((c_f_non - gt_non)**2)
        L_non_occ = loss_c_non + loss_f_non
    else:
        L_non_occ = 0.0
        
    # Compute L_Occ
    # Rays IN occluded region
    loss_c_occ = torch.mean((coarse_colors - prior_vals)**2)
    loss_f_occ = torch.mean((fine_colors - prior_vals)**2)
    L_occ = loss_c_occ + loss_f_occ
    
    # Total Loss
    total_loss = lambda_val * L_non_occ + (1 - lambda_val) * L_occ
    
    return total_loss

def compute_average_gradient(image: Tensor, mask: Tensor) -> float:
    """
    Computes the average gradient magnitude over masked pixels.
    """
    # Apply Sobel filters or similar to get gradient magnitude
    grad_x = sobel_x(image)
    grad_y = sobel_y(image)
    grad_mag = torch.sqrt(grad_x**2 + grad_y**2)
    
    # Mask the gradient magnitude
    masked_grad = grad_mag * mask
    
    # Sum of gradients in masked region
    sum_grad = torch.sum(masked_grad)
    # Number of pixels in masked region
    num_pixels = torch.sum(mask)
    
    if num_pixels == 0:
        return 0.0
        
    return sum_grad / num_pixels
```

**Parameters:**
*   $\tau$: Threshold for gradient comparison. Not given in the sources. [G: Typical values for such thresholds in image processing range from 0.1 to 1.0, but must be tuned per dataset.]
*   Sub-region size: Determines the granularity of gradient evaluation. Not given in the sources. [G: Common patch sizes are 8x8 or 16x16 pixels.]

**Edge Cases and Pitfalls:**
*   **Uniform Strategy Misjudgment**: Applying a uniform strategy to the entire occluded area can lead to misjudgment. The method avoids this by dividing the occluded area into sub-regions and evaluating gradients individually [S1].
*   **2D Inpainter Disturbance**: Single 2D inpainting does not consider 3D consistency. Relying on it when multi-view information is available (high gradient in NeRF) can disturb the 3D editing [S1].
*   **Lack of Supplementary Information**: If no other views can provide supplementary information (e.g., deep occlusion), the method relies on the 2D inpainter ($\lambda=0$) [S1].

## Operating characteristics
*   **Time Complexity**: Depends on the number of rays, the size of the occluded region, and the cost of the 2D inpainter (LaMa). The gradient calculation adds overhead proportional to the number of pixels in the occluded region.
*   **Memory Complexity**: Requires storing the NeRF model, the 2D inpainter model, and intermediate rendered images for gradient calculation.
*   **Strengths**: Improves fidelity of de-occluded results by leveraging multi-view correlation. Avoids the inconsistency of 2D inpainting in 3D scenes.
*   **Limitations**: Relies on the quality of the 2D inpainter (LaMa) and the accuracy of the object segmentation network for mask generation. Performance depends on the availability of multi-view information.
*   **When to use**: When inpainting occluded areas in NeRFs where multi-view consistency is crucial and 2D inpainting alone leads to artifacts.
*   **When not to use**: When the occluded area is completely hidden from all other views and the 2D inpainter is unreliable, or when computational resources are too limited for the additional gradient calculations.

## Background
The method originates from the paper "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1]. It was developed to address the limitations of existing NeRF inpainting methods that either ignored 3D consistency or relied solely on 2D priors. The framework integrates a region coding network for segmentation and a 3D inpainting component with adaptive view selection.

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
