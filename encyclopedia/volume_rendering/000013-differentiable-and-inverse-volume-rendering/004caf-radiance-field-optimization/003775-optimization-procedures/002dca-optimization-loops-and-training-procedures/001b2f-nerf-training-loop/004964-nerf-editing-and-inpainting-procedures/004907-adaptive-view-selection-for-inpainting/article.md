# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
Adaptive View Selection for Inpainting is a method for 3D scene de-occlusion and inpainting that combines a pretrained 2D image inpainter with multi-view NeRF optimization [S1]. It solves the problem of inconsistent texture synthesis in occluded regions by dynamically selecting whether to rely on 2D priors or multi-view geometric consistency based on local texture richness [S1]. Unlike standard NeRF editing which may propagate 2D inpainting artifacts across views, this method uses a regional average gradient metric to adaptively weight the reconstruction loss, ensuring that high-texture areas leverage multi-view correlation while low-texture areas rely on the 2D prior [S1].

## How it works
The method operates within a NeRF framework that includes a coarse network ($\hat{C}_c$) and a fine network ($\hat{C}_f$) [S1]. It requires an initial object segmentation mask to identify occluded regions and uses a pretrained 2D inpainter (specifically LaMa [S1]) to generate appearance priors for these regions [S1].

### Inputs and Outputs
**Inputs:**
*   A set of input views $V$ with corresponding images and camera poses.
*   A binary mask $M$ identifying the occluded region in at least one view (typically derived from an object segmentation network [S1]).
*   A pretrained 2D inpainter (e.g., LaMa) to generate inpainted images $I_{inpaint}$ for the occluded regions [S1].
*   Ground truth depth $D$ for the scene (optional, used in joint optimization [S1]).

**Outputs:**
*   Optimized NeRF parameters (MLP weights for $\sigma$ and $\mathbf{c}$).
*   Inpainted novel views with consistent texture across occluded boundaries [S1].

### Data Structures
*   **NeRF Model:** Two MLPs (coarse and fine) mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
*   **Predicted Mask $\hat{P}(r)$:** A scalar value per ray $r$, indicating the probability that the ray passes through the occluded region. This is typically output by a region coding network or derived from the initial segmentation mask projected into 3D [S1].
*   **Inpainting Prior $C_l(r)$:** The color value from the 2D inpainted image corresponding to ray $r$ [S1].

### Numbered Steps

1.  **Initial 2D Inpainting:** For each input view containing occluded regions, apply the pretrained 2D inpainter to generate a complete image $I_{inpaint}$. Extract the color prior $C_l(r)$ for rays $r$ passing through the occluded mask $\hat{P}(r)=1$ [S1].

2.  **Region Coding and Masking:** Use a region coding network (or projected segmentation) to generate the predicted mask $\hat{P}(r)$ for all rays. This mask distinguishes between non-occluded ($\hat{P}(r) \approx 0$) and occluded ($\hat{P}(r) \approx 1$) areas [S1].

3.  **Sub-region Division:** Divide the occluded area in the current view into multiple sub-regions (patches) [S1]. This allows for localized evaluation of texture richness rather than a global uniform strategy [S1].

4.  **Gradient Evaluation:** For each sub-region, calculate the average gradient of the rendered result $\nabla \hat{g}$ and the average gradient of the 2D inpainting result $\nabla g$ [S1].
    *   $\nabla \hat{g}$: Average gradient magnitude of the NeRF-rendered image in the sub-region.
    *   $\nabla g$: Average gradient magnitude of the 2D inpainted image in the sub-region [S1].

5.  **Adaptive Weight Calculation:** Compute the adaptive weight $\lambda$ for the sub-region based on the threshold $\tau$ [S1]:
    $$
    \lambda = \begin{cases} 
    1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
    0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g 
    \end{cases}
    $$
    *   If $\lambda = 1$, the rendered texture is richer than the 2D prior, implying multi-view consistency is achievable. The loss relies on ground truth (if available) or multi-view consistency.
    *   If $\lambda = 0$, the rendered texture is poor, and the method relies on the 2D inpainting prior $C_l(r)$ [S1].

6.  **Loss Computation:** Calculate the total reconstruction loss $L$ as a weighted sum of non-occluded loss $L_{NonOcc}$ and occluded loss $L_{Occ}$ [S1]:
    $$
    L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ}
    $$
    
    *   **Non-occluded Loss ($L_{NonOcc}$):** Sum over rays $r \in R$ where $\hat{P}(r) \approx 0$:
        $$
        L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ \|\hat{C}_c(r) - C(r)\|_2^2 + \|\hat{C}_f(r) - C(r)\|_2^2 \right]
        $$
        Where $C(r)$ is the ground truth color [S1].
        
    *   **Occluded Loss ($L_{Occ}$):** Sum over rays $r \in R$ where $\hat{P}(r) \approx 1$:
        $$
        L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ \|\hat{C}_c(r) - C_l(r)\|_2^2 + \|\hat{C}_f(r) - C_l(r)\|_2^2 \right]
        $$
        Where $C_l(r)$ is the color from the 2D inpainted image [S1].

7.  **Joint Optimization:** Update NeRF parameters using gradient descent on $L$. Optionally, include a depth loss $L_{depth} = \sum_{r \in R} \|\hat{D}(r) - D(r)\|_2^2$ and region encoding loss to ensure multi-view consistency of the segmentation mask [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_view_selection_inpainting(nerf_coarse, nerf_fine, 
                                      input_views, masks_2d, 
                                      inpainter_model, tau, 
                                      sub_region_size):
    """
    Performs 3D inpainting with adaptive view selection.
    
    Args:
        nerf_coarse, nerf_fine: NeRF MLPs for coarse and fine rendering.
        input_views: List of dicts with 'image', 'camera_pose', 'mask'.
        masks_2d: Binary masks for occluded regions in input_views.
        inpainter_model: Pretrained 2D inpainter (e.g., LaMa).
        tau: Threshold for gradient comparison.
        sub_region_size: Size of patches for gradient evaluation.
        
    Returns:
        Optimized nerf_coarse, nerf_fine.
    """
    
    # Step 1: Generate 2D inpainting priors
    inpainted_images = []
    for view in input_views:
        # Apply 2D inpainter to the occluded region defined by view['mask']
        inpainted_img = inpainter_model(view['image'], view['mask'])
        inpainted_images.append(inpainted_img)
        
    # Step 2: Initialize/Update Region Coding Network to get predicted mask P_hat
    # Note: In practice, P_hat might be a separate network or derived from 
    # multi-view consistency constraints. Here we assume it's available 
    # or updated via a separate loss component [S1].
    region_coding_net = initialize_region_coding_network()
    
    optimizer = torch.optim.Adam(list(nerf_coarse.parameters()) + 
                                 list(nerf_fine.parameters()) + 
                                 list(region_coding_net.parameters()), lr=1e-4)
    
    for epoch in range(num_epochs):
        total_loss = 0.0
        
        for view_idx, view in enumerate(input_views):
            rays = generate_rays(view['camera_pose'], view['image'].shape)
            
            # Render current NeRF estimate
            colors_c, colors_f, depths = render_nerf(nerf_coarse, nerf_fine, rays)
            
            # Get predicted mask for these rays
            # P_hat is a scalar per ray indicating occlusion probability [S1]
            P_hat = region_coding_net(rays.positions, rays.directions)
            
            # Get ground truth color C(r)
            C_gt = sample_colors_from_image(view['image'], rays)
            
            # Get 2D inpainting prior C_l(r)
            # Map rays to pixels in the inpainted image
            C_l = sample_colors_from_image(inpainted_images[view_idx], rays)
            
            # Step 3 & 4: Divide into sub-regions and evaluate gradients
            # This is done per sub-region to calculate lambda locally
            # For simplicity, we aggregate the loss per sub-region
            
            sub_regions = divide_into_patches(rays, P_hat, sub_region_size)
            
            loss_non_occ = 0.0
            loss_occ = 0.0
            
            for region in sub_regions:
                rays_in_region = region.rays
                P_hat_region = region.P_hat
                C_gt_region = region.C_gt
                C_l_region = region.C_l
                
                # Render colors for this region
                colors_c_reg, colors_f_reg, _ = render_nerf(nerf_coarse, nerf_fine, rays_in_region)
                
                # Calculate average gradients
                # grad_hat: average gradient of rendered image in this region
                grad_hat = compute_average_gradient(colors_c_reg, colors_f_reg)
                
                # grad: average gradient of 2D inpainted image in this region
                grad = compute_average_gradient(C_l_region)
                
                # Step 5: Adaptive weight lambda
                if grad_hat > tau * grad:
                    lambda_val = 1.0
                else:
                    lambda_val = 0.0
                
                # Step 6: Compute losses for this region
                # Non-occluded loss component (weighted by 1 - P_hat)
                # Note: In occluded regions, P_hat is high, so (1-P_hat) is low.
                # However, the formula [S1] uses (1-P_hat) for non-occluded loss.
                # We must ensure we don't double count or misapply.
                # The total loss L = lambda * L_NonOcc + (1-lambda) * L_Occ
                
                # L_NonOcc part for this region
                L_non_occ_reg = torch.mean((1 - P_hat_region) * 
                                           (torch.norm(colors_c_reg - C_gt_region, p=2)**2 + 
                                            torch.norm(colors_f_reg - C_gt_region, p=2)**2))
                
                # L_Occ part for this region
                L_occ_reg = torch.mean(P_hat_region * 
                                       (torch.norm(colors_c_reg - C_l_region, p=2)**2 + 
                                        torch.norm(colors_f_reg - C_l_region, p=2)**2))
                
                # Accumulate weighted loss for this region
                # Note: lambda is calculated per region, but applied to the 
                # global loss structure. The paper suggests lambda modulates 
                # the contribution of L_NonOcc vs L_Occ.
                # If lambda=1, we trust the NeRF (L_NonOcc style, i.e., GT).
                # If lambda=0, we trust the Prior (L_Occ style, i.e., C_l).
                
                # However, L_NonOcc is defined for non-occluded areas.
                # The paper's formula (6) is: L = lambda * L_NonOcc + (1-lambda) * L_Occ
                # This implies lambda is a global or regional switch.
                # If regional, we sum the regional contributions.
                
                total_loss += lambda_val * L_non_occ_reg + (1 - lambda_val) * L_occ_reg

        # Optional: Add depth loss and region encoding loss [S1]
        # L_depth = sum(||D_hat - D||^2)
        # L_region = region_encoding_loss
        
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        
    return nerf_coarse, nerf_fine

def compute_average_gradient(colors):
    """Computes the average gradient magnitude of an image tensor."""
    # Convert to grayscale if necessary
    if len(colors.shape) == 4: # B, H, W, C
        gray = torch.mean(colors, dim=-1)
    else:
        gray = colors
        
    # Sobel filters
    sobel_x = torch.tensor([[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]], dtype=colors.dtype).unsqueeze(0).unsqueeze(0)
    sobel_y = torch.tensor([[-1, -2, -1], [0, 0, 0], [1, 2, 1]], dtype=colors.dtype).unsqueeze(0).unsqueeze(0)
    
    # Convolve
    grad_x = torch.nn.functional.conv2d(gray, sobel_x, padding=1)
    grad_y = torch.nn.functional.conv2d(gray, sobel_y, padding=1)
    
    magnitude = torch.sqrt(grad_x**2 + grad_y**2 + 1e-6)
    return torch.mean(magnitude)
```

### Parameters
*   $\tau$: Threshold for gradient comparison. Not given in the sources. [G: Typical values for such thresholds in image processing range from 0.1 to 1.0, but must be tuned per dataset.]
*   $C_l(r)$: Color of the corresponding pixel in the 2D inpainting image [S1].
*   $\hat{C}_c(r), \hat{C}_f(r)$: Predicted colors from coarse and fine NeRF networks [S1].
*   $C(r)$: Ground truth color [S1].

### Edge Cases and Pitfalls
*   **Uniform Strategy Misjudgment:** Applying a single global weight to the entire occluded area can lead to poor results if some parts have rich texture (benefiting from multi-view) and others are uniform (benefiting from 2D prior). The method mitigates this by dividing the occluded area into sub-regions [S1].
*   **2D Inpainter Artifacts:** Single 2D inpainting does not consider 3D consistency. Relying on it too heavily in high-texture regions causes flickering or inconsistency across views. The gradient-based selection prevents this [S1].
*   **Precision:** The gradient calculation must be robust to noise. Small variations in $\nabla \hat{g}$ near the threshold $\tau$ can cause instability. [G: Smoothing the gradient map or using a soft threshold (e.g., sigmoid) might be necessary in implementation, though the source specifies a hard threshold.]

## Operating characteristics
*   **Time Complexity:** Depends on the number of NeRF training iterations and the cost of the 2D inpainter. The sub-region gradient calculation adds overhead proportional to the number of sub-regions.
*   **Memory Complexity:** Requires storing the 2D inpainted images and the predicted mask $\hat{P}$ for all rays.
*   **Strengths:** Effectively reduces disturbing artifacts from 2D inpainters by leveraging multi-view consistency where texture is rich [S1]. Improves fidelity of de-occluded results [S1].
*   **Limitations:** Relies on the quality of the 2D inpainter (LaMa) and the accuracy of the initial segmentation mask [S1]. Performance depends on the availability of multi-view information; if no other views see the occluded area, it falls back to the 2D prior [S1].
*   **When to use:** When inpainting occluded regions in NeRFs where multi-view consistency is critical and the occluded area has varying texture complexity.
*   **When not to use:** If the scene has very few views or if the occluded region is completely invisible in all other views (though the method handles this by falling back to 2D prior, the benefit of adaptive selection is reduced).

## Background
The method was proposed in "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1]. It addresses the limitation of standard NeRF editing methods that do not account for 3D consistency when using 2D priors. The core innovation is the training view selection mechanism based on regional average gradients [S1].

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
