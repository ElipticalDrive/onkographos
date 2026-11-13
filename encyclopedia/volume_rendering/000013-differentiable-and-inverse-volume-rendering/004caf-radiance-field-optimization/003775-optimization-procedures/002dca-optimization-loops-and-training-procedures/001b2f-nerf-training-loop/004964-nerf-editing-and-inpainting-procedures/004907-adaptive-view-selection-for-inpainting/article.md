# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
Adaptive View Selection for Inpainting is a method for 3D scene de-occlusion and object removal that combines a pretrained 2D image inpainter with multi-view NeRF optimization [S1]. It solves the inconsistency problem of applying 2D inpainting directly to 3D scenes by selectively using 2D inpainting priors only when multi-view geometric consistency cannot be established [S1]. Unlike sibling methods that rely on generative diffusion priors or feature distillation, this approach uses a gradient-based view selection mechanism to dynamically weight the reconstruction loss between standard photometric error and 2D inpainting priors [S1].

## How it works
The method operates on a NeRF representation $f_\theta$ mapping $(\mathbf{x}, \mathbf{d})$ to density $\sigma$ and color $\mathbf{c}$, typically using a coarse and fine network architecture ($\hat{C}_c, \hat{C}_f$) [S1]. It requires a set of input views, ground truth masks for occluded regions, and a pretrained 2D inpainter (specifically LaMa) [S1].

### Data Structures
*   **NeRF Networks**: Two MLPs (coarse and fine) taking 3D position $\mathbf{x}$ and direction $\mathbf{d}$ as input, outputting $\sigma$ and $\mathbf{c}$ [S1].
*   **Segmentation Mask $\hat{P}(r)$**: A predicted binary mask for ray $r$, where $\hat{P}(r)=1$ indicates the ray passes through an occluded area and $\hat{P}(r)=0$ indicates a non-occluded area [S1].
*   **2D Inpainting Prior $C_l(r)$**: The color value of the pixel corresponding to ray $r$ after being processed by the 2D inpainter (LaMa) [S1].
*   **Sub-regions**: The occluded area in each view is divided into multiple sub-regions for gradient evaluation [S1].

### Parameters
*   $\tau$: Threshold multiplier for gradient comparison. Not given in the sources.
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for the segmentation network losses (region encoding, RGB, depth). Not given in the sources.

### Steps
1.  **Initial 2D Inpainting**: Apply the pretrained 2D inpainter (LaMa) to all input views to generate initial appearance priors $C_l$ for occluded areas [S1].
2.  **Segmentation**: Train an object segmentation network to predict masks $\hat{P}(r)$ for occluded areas across views, using a joint loss $L = \lambda_1 L_{region-encoding} + \lambda_2 L_{rgb} + \lambda_3 L_{depth}$ to ensure multi-view consistency [S1].
3.  **Sub-region Division**: Divide the occluded area in each view into multiple sub-regions [S1].
4.  **Gradient Evaluation**: Calculate the average gradient for each sub-region in the rendered result ($\nabla \hat{g}$) and the 2D inpainting result ($\nabla g$) [S1].
5.  **Adaptive Weight Calculation**: Compute the adaptive weight $\lambda$ for each sub-region based on the gradient comparison [S1]:
    $$
    \lambda = \begin{cases} 
    1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
    0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g 
    \end{cases}
    $$
    If $\nabla \hat{g}$ is high, the NeRF rendering has sufficient texture detail, so the method relies on multi-view consistency ($\lambda=1$, using $L_{NonOcc}$). If $\nabla \hat{g}$ is low, the NeRF rendering lacks detail, so the method falls back to the 2D inpainting prior ($\lambda=0$, using $L_{Occ}$) [S1].
6.  **Loss Computation**: Compute the total reconstruction loss $L$ using the adaptive weight $\lambda$ [S1]:
    $$
    L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ}
    $$
    Where:
    *   $L_{NonOcc}$ (Non-occluded loss):
        $$
        L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ ||\hat{C}_c(r) - C(r)||_2^2 + ||\hat{C}_f(r) - C(r)||_2^2 \right]
        $$
        Here, $C(r)$ is the ground truth color [S1].
    *   $L_{Occ}$ (Occluded loss):
        $$
        L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ ||\hat{C}_c(r) - C_l(r)||_2^2 + ||\hat{C}_f(r) - C_l(r)||_2^2 \right]
        $$
        Here, $C_l(r)$ is the color from the 2D inpainting prior [S1].
7.  **Optimization**: Update NeRF parameters $\theta$ and segmentation network parameters to minimize $L$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_adaptive_loss(rendered_coarse, rendered_fine, ground_truth, 
                          occlusion_mask, inpainting_prior, sub_regions, 
                          tau, threshold_multiplier):
    """
    Computes the adaptive reconstruction loss for NeRF inpainting.
    
    Args:
        rendered_coarse: Tensor of coarse predicted colors for rays R.
        rendered_fine: Tensor of fine predicted colors for rays R.
        ground_truth: Tensor of ground truth colors for rays R.
        occlusion_mask: Tensor of predicted occlusion masks P_hat for rays R.
        inpainting_prior: Tensor of 2D inpainted colors C_l for rays R.
        sub_regions: List of sub-region indices or masks for the occluded area.
        tau: Threshold multiplier for gradient comparison.
        threshold_multiplier: Scalar tau value.
        
    Returns:
        Total loss L.
    """
    # Step 1: Calculate gradients for non-occluded areas (for reference)
    # Note: The source defines lambda based on gradients of the *occluded* area 
    # rendered result vs inpainting result.
    
    # Step 2: Evaluate gradients for each sub-region in the occluded area
    # We assume a function to get average gradient of a tensor region
    def get_avg_gradient(tensor, mask):
        # Compute gradient magnitude (e.g., Sobel) then average over masked region
        grad_mag = compute_gradient_magnitude(tensor)
        masked_grad = grad_mag * mask
        return torch.mean(masked_grad)

    # Initialize adaptive weight lambda
    # In practice, lambda might be computed per sub-region or globally for the view.
    # The source implies a "refined patching method" evaluating sub-regions individually.
    # For a single scalar lambda for the whole loss term as per Eq 6:
    
    # Get rendered gradient in occluded area
    occluded_rendered_grad = get_avg_gradient(rendered_coarse, occlusion_mask)
    
    # Get inpainting gradient in occluded area
    occluded_inpaint_grad = get_avg_gradient(inpainting_prior, occlusion_mask)
    
    # Determine lambda
    if occluded_rendered_grad > threshold_multiplier * occluded_inpaint_grad:
        lambda_val = 1.0
    else:
        lambda_val = 0.0
        
    # Step 3: Compute L_NonOcc
    # Mask for non-occluded rays
    non_occluded_mask = 1.0 - occlusion_mask
    
    # L2 loss for coarse and fine networks against ground truth
    loss_coarse_nonocc = torch.mean(non_occluded_mask * (rendered_coarse - ground_truth)**2)
    loss_fine_nonocc = torch.mean(non_occluded_mask * (rendered_fine - ground_truth)**2)
    
    L_nonocc = loss_coarse_nonocc + loss_fine_nonocc
    
    # Step 4: Compute L_Occ
    # L2 loss for coarse and fine networks against inpainting prior
    loss_coarse_occ = torch.mean(occlusion_mask * (rendered_coarse - inpainting_prior)**2)
    loss_fine_occ = torch.mean(occlusion_mask * (rendered_fine - inpainting_prior)**2)
    
    L_occ = loss_coarse_occ + loss_fine_occ
    
    # Step 5: Combine with adaptive weight
    L = lambda_val * L_nonocc + (1.0 - lambda_val) * L_occ
    
    return L
```

### Edge Cases and Pitfalls
*   **Uniform Strategy Misjudgment**: Applying a single strategy to the entire occluded area can fail if some parts have rich texture (visible in other views) and others do not. The method mitigates this by dividing the occluded area into sub-regions and evaluating gradients individually [S1].
*   **2D Inpainter Inconsistency**: Single 2D inpainting does not consider 3D consistency. The method addresses this by using multi-view correlation to decide when to trust the 2D inpainter [S1].
*   **Gradient Threshold**: The choice of $\tau$ is critical. If too low, the method may rely too heavily on inconsistent 2D priors. If too high, it may ignore useful 2D priors in textureless regions. Not given in the sources.

## Operating characteristics
*   **Time Complexity**: Depends on the NeRF training time and the 2D inpainting inference time. The gradient calculation adds minimal overhead. Not given in the sources.
*   **Memory Complexity**: Requires storing coarse and fine NeRF networks, segmentation network, and intermediate gradients. Not given in the sources.
*   **Strengths**: Effectively reduces the disturbing effect of 2D inpainters by leveraging multi-view consistency where available [S1]. Handles scenes with a wide range of multi-view coverage [S1].
*   **Limitations**: Relies on the quality of the pretrained 2D inpainter (LaMa). Performance depends on the accuracy of the segmentation mask $\hat{P}(r)$ [S1].
*   **When to use**: When inpainting occluded areas in NeRFs and multi-view consistency is available for some parts of the occlusion.
*   **When not to use**: When no multi-view information is available for the occluded area (though the method falls back to 2D inpainting, the benefit of adaptive selection is reduced) [S1].

## Background
The method was proposed in "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1]. It addresses the limitation of previous 3D inpainting methods that either relied solely on 2D inpainting (ignoring 3D consistency) or optimization-based methods that struggled with textureless regions. The method builds on the NeRF rendering equation [S1].

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
