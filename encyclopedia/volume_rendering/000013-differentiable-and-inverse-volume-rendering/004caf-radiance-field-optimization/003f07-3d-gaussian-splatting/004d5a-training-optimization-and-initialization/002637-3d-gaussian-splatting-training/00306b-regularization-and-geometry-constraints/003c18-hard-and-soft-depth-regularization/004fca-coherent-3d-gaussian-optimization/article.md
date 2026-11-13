# Coherent 3D Gaussian Optimization

Also called: CoherentGS

## Overview
Coherent 3D Gaussian Optimization is a regularization strategy for 3D Gaussian Splatting that enforces spatial coherence in Gaussian positions and opacities during training, specifically addressing artifacts in sparse-view novel view synthesis. It sets itself apart from standard 3DGS by replacing independent per-pixel residual updates with a learned implicit decoder that produces smooth, coherent residual fields for depth and opacity, supplemented by multiview total variation and optical flow constraints. This prevents the "semi-transparent" artifacts and detail loss common when optimizing Gaussians against sparse input views.

## How it works
The method optimizes the Gaussian parameters ($\Sigma$, $\mathbf{c}$) indirectly by optimizing the parameters of an implicit convolutional decoder ($\phi$) that generates residual corrections for initial depth and opacity estimates.

**Inputs and Outputs:**
*   **Inputs:**
    *   Set of input views $V$.
    *   Initial depth maps $D^{\text{init}}_n$ for each view $n$, obtained via monocular depth estimation (e.g., Yang et al. [60]).
    *   Initial Gaussian parameters $\Sigma, \mathbf{c}$ (from standard 3DGS initialization).
    *   Camera poses for each view.
*   **Outputs:**
    *   Optimized Gaussian parameters $\Sigma^*, \mathbf{c}^*$.
    *   Optimized decoder parameters $\phi^*$.

**Data Structures:**
*   **Implicit Decoder $f_\phi$:** A convolutional neural network taking the normalized view index $n$ as input. It outputs a tensor of residual values.
*   **Segmentation Mask $S$:** A binary mask of shape $(C, H, W)$, where $C=5$ is the number of regions. Regions are determined by dividing the image based on the input monocular depth $D^{\text{init}}_n$ using the approach by Wang et al. [53].
*   **Optical Flow Correspondences:** Pairs of pixels $(\mathbf{p}, \mathbf{q})$ between views $i$ and $j$ computed via optical flow (Shi et al. [45]).
*   **Forward-Backward Consistency Mask $M_{i \rightarrow j}$:** A binary mask indicating reliable pixel correspondences between views $i$ and $j$.

**Numbered Steps:**

1.  **Initialization:**
    *   Compute initial depth maps $D^{\text{init}}_n$ for all input views using a monocular depth estimator [G: Yang et al.].
    *   Initialize Gaussian parameters $\Sigma, \mathbf{c}$ using standard 3DGS initialization.
    *   Initialize decoder parameters $\phi$ randomly.
    *   Initialize segmentation masks $S_n$ for each view by dividing the image into $C=5$ regions based on $D^{\text{init}}_n$ [G: Wang et al.].

2.  **Residual Depth Estimation (Single-View Constraint):**
    *   For each view $n$, pass the normalized view index $n$ to the decoder $f_\phi(n)$.
    *   The decoder outputs a $C$-channel residual depth tensor.
    *   Compute the final residual depth $\Delta D_n$ by applying channelwise summation (`chsum`) of the element-wise product of the segmentation mask $S_n$ and the decoder output:
        $$ \Delta D_n = \text{chsum}(S_n \odot f_\phi(n)) $$
    *   This ensures smooth deformation within regions while allowing discontinuities between regions.

3.  **Residual Opacity Estimation:**
    *   Similarly, estimate residual opacity using a decoder (or shared architecture) producing $C$-channel residual opacity.
    *   Combine with initial opacity to obtain final opacity at each pixel.

4.  **Gaussian Position Update:**
    *   For each pixel $\mathbf{p}$ in view $n$, compute the updated 3D position $\mathbf{x}$ by projecting the residual depth into 3D space:
        $$ \mathbf{x} = g(D^{\text{init}}_n [\mathbf{p}] + \Delta D_n [\mathbf{p}], \mathbf{p}) $$
    *   Here, $g(d, \mathbf{p})$ is the function that projects pixel $\mathbf{p}$ into 3D space according to depth $d$.
    *   The Gaussian centers are updated to these positions $\mathbf{x}$.

5.  **Multiview Total Variation Loss ($\mathcal{L}_{\text{multi}}$):**
    *   Render the depth map $R_{\Sigma, \alpha, \mathbf{x}, d}$ for each view using the current Gaussian parameters and updated positions.
    *   Compute the rendered disparity as $1 / (1 + R_{\Sigma, \alpha, \mathbf{x}, d})$.
    *   Compute the global TV loss:
        $$ \mathcal{L}_{\text{TV}} = \left\| \nabla \left( \frac{1}{1 + R_{\Sigma, \alpha, \mathbf{x}, d}} \right) \right\|_1 $$
    *   Compute the masked TV loss using the segmentation mask $S$:
        $$ \mathcal{L}_{\text{MTV}} = \left\| \nabla \left( S \odot \left( \frac{1}{1 + R_{\Sigma, \alpha, \mathbf{x}, d}} \right) \right) \right\|_1 $$
    *   Combine them with a schedule parameter $\lambda_s$:
        $$ \mathcal{L}_{\text{multi}} = (1 - \lambda_s)\mathcal{L}_{\text{TV}} + \lambda_s\mathcal{L}_{\text{MTV}} $$
    *   $\lambda_s$ is initialized to 0 and gradually increased to 1 by the end of optimization.

6.  **Flow Loss ($\mathcal{L}_{\text{flow}}$):**
    *   For each pair of views $(i, j)$ with corresponding pixels $(\mathbf{p}, \mathbf{q})$ identified by optical flow:
    *   Compute the 3D positions for both pixels using their respective residual depths:
        $$ D_i [\mathbf{p}] = D^{\text{init}}_i [\mathbf{p}] + \Delta D_i [\mathbf{p}] $$
        $$ D_j [\mathbf{q}] = D^{\text{init}}_j [\mathbf{q}] + \Delta D_j [\mathbf{q}] $$
        $$ \mathbf{x}_i = g(D_i [\mathbf{p}], \mathbf{p}), \quad \mathbf{x}_j = g(D_j [\mathbf{q}], \mathbf{q}) $$
    *   Compute the flow loss using the consistency mask $M_{i \rightarrow j}$:
        $$ \mathcal{L}_{\text{flow}} = \sum_{(i, j)} \sum_{\mathbf{p}} \left\| M_{i \rightarrow j} \odot \left( \mathbf{x}_i - \mathbf{x}_j \right) \right\|_1 $$

7.  **Multisampling:**
    *   During rendering for color loss calculation, evaluate multiple samples within each pixel and average them to obtain the pixel color. This ensures Gaussians properly cover the pixel area, preventing semi-transparency in novel views.

8.  **Optimization Step:**
    *   Minimize the total objective function with respect to $\Sigma, \phi, \mathbf{c}$:
        $$ \Sigma^*, \phi^*, \mathbf{c}^* = \arg \min_{\Sigma, \phi, \mathbf{c}} \sum_{\mathbf{p} \in \mathcal{P}} \mathcal{L}(R_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}}(\mathbf{p}), R(\mathbf{p})) + \beta_m \mathcal{L}_{\text{multi}} + \beta_f \mathcal{L}_{\text{flow}} $$
    *   Typical weights: $\beta_m = 5$, $\beta_f = 0.1$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coherent_3dgs_optimization(input_views, initial_depths, initial_gaussians, decoder, num_regions=5):
    """
    Optimizes 3D Gaussians using coherent regularization.
    
    Args:
        input_views: List of view indices and camera poses.
        initial_depths: Dict mapping view index to initial depth map D_init.
        initial_gaussians: Initial Gaussian parameters (Sigma, c).
        decoder: Implicit convolutional decoder f_phi.
        num_regions: Number of segmentation regions C.
        
    Returns:
        Optimized Gaussian parameters Sigma*, c*, and decoder parameters phi*.
    """
    # Initialize parameters
    phi = decoder.parameters()
    Sigma = initial_gaussians.Sigma
    c = initial_gaussians.c
    
    # Precompute segmentation masks for each view
    # Note: In practice, this might be done per-view during training or precomputed
    segmentation_masks = {}
    for view_idx in input_views:
        D_init = initial_depths[view_idx]
        # Divide image into C regions based on D_init
        # Implementation detail: Wang et al. [53] strategy
        segmentation_masks[view_idx] = compute_segmentation_mask(D_init, num_regions)
        
    # Precompute optical flow correspondences and consistency masks
    # This is expensive, so typically done once or periodically
    flow_correspondences = {}
    consistency_masks = {}
    for i, view_i in enumerate(input_views):
        for j, view_j in enumerate(input_views):
            if i >= j: continue
            # Compute optical flow between view_i and view_j
            # Shi et al. [45]
            flow = compute_optical_flow(view_i, view_j)
            # Compute forward-backward consistency mask
            # Wang et al. [32]
            mask = compute_consistency_mask(flow)
            flow_correspondences[(view_i, view_j)] = flow
            consistency_masks[(view_i, view_j)] = mask

    # Optimization loop
    lambda_s = 0.0
    for epoch in range(num_epochs):
        # Update lambda_s schedule
        lambda_s = min(1.0, epoch / num_epochs)
        
        for view_idx in input_views:
            # 1. Get residual depth from decoder
            # Decoder takes normalized view index
            normalized_view_idx = normalize_view_index(view_idx)
            residual_depth_tensor = decoder(normalized_view_idx) # Shape: (C, H, W)
            
            # 2. Apply segmentation mask and channelwise summation
            S = segmentation_masks[view_idx] # Shape: (C, H, W)
            residual_depth = channelwise_sum(S * residual_depth_tensor) # Shape: (H, W)
            
            # 3. Compute updated depth and 3D positions
            D_init = initial_depths[view_idx]
            updated_depth = D_init + residual_depth
            
            # Project pixels to 3D
            # g(d, p) projects pixel p into 3D according to depth d
            positions = []
            for p in all_pixels:
                x = project_pixel_to_3d(p, updated_depth[p], camera_pose[view_idx])
                positions.append(x)
            
            # 4. Render depth and color
            # Render depth map R_d and color map R_c
            R_d = render_depth(Sigma, alpha, positions, camera_pose[view_idx])
            R_c = render_color(Sigma, alpha, positions, c, camera_pose[view_idx], multisample=True)
            
            # 5. Compute TV Losses
            disparity = 1.0 / (1.0 + R_d)
            grad_disparity = compute_gradient(disparity)
            L_TV = l1_norm(grad_disparity)
            
            masked_disparity = S * disparity
            grad_masked_disparity = compute_gradient(masked_disparity)
            L_MTV = l1_norm(grad_masked_disparity)
            
            L_multi = (1 - lambda_s) * L_TV + lambda_s * L_MTV
            
            # 6. Compute Flow Loss
            L_flow = 0.0
            for (view_i, view_j), flow in flow_correspondences.items():
                if view_i == view_idx or view_j == view_idx:
                    # Compute positions for corresponding pixels
                    # Note: This requires iterating over all corresponding pixels
                    # For efficiency, this is often computed in a batched manner
                    pass 
                # ... (Flow loss computation as per formula)
            
            # 7. Compute Color Loss
            # Reference color R(p) from input view
            L_color = compute_color_loss(R_c, reference_color[view_idx])
            
            # 8. Total Loss
            beta_m = 5.0
            beta_f = 0.1
            total_loss = L_color + beta_m * L_multi + beta_f * L_flow
            
            # 9. Backpropagate and update
            total_loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            
    return Sigma, c, phi
```

**Parameters:**
*   $C=5$: Number of segmentation regions.
*   $\beta_m = 5$: Weight for multiview TV loss.
*   $\beta_f = 0.1$: Weight for flow loss.
*   $\lambda_s$: Schedule parameter for TV loss, starting at 0 and ending at 1.

**Edge Cases and Pitfalls:**
*   **Sparse Views:** Standard center-sampling leads to semi-transparency. Multisampling is required.
*   **Sharp Discontinuities:** Smooth decoder alone fails. Segmentation masks are necessary to handle object boundaries.
*   **Optical Flow Errors:** Unreliable correspondences must be masked out using forward-backward consistency.

## Operating characteristics
*   **Time Complexity:** Higher than standard 3DGS due to the additional decoder forward pass, segmentation mask computation, and flow loss calculation. The decoder parameters $\phi$ are optimized alongside Gaussians.
*   **Memory Complexity:** Requires storing segmentation masks and optical flow correspondences. The decoder adds minimal memory overhead.
*   **Performance:** Improves novel view synthesis quality in sparse-view settings by reducing artifacts and improving geometry coherence.
*   **Strengths:** Handles sparse inputs better than standard 3DGS; enforces smooth geometry; prevents semi-transparency.
*   **Limitations:** Requires monocular depth estimation for initialization; relies on optical flow which can be noisy; computationally more expensive.
*   **When to use:** Sparse-view novel view synthesis where standard 3DGS produces blurry or transparent artifacts.
*   **When not to use:** Dense-view settings where standard 3DGS performs well; real-time applications where the additional computational cost is prohibitive.

## Background
Coherent 3D Gaussian Optimization was introduced in "CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians" by A. Paliwal et al. [S1]. It builds on the 3D Gaussian Splatting framework [S2] and addresses its limitations in sparse-view scenarios. The method incorporates ideas from consistent depth estimation [S3, S4] and optical flow [S5].

## Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
[S2] 3D Gaussian Splatting, https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/, fetched 2026-10-27
[S3] Consistent Depth Estimation, https://arxiv.org/abs/1904.02089, fetched 2026-10-27
[S4] Forward-Backward Consistency, https://arxiv.org/abs/1504.01685, fetched 2026-10-27
[S5] Optical Flow, https://arxiv.org/abs/1706.02515, fetched 2026-10-27
