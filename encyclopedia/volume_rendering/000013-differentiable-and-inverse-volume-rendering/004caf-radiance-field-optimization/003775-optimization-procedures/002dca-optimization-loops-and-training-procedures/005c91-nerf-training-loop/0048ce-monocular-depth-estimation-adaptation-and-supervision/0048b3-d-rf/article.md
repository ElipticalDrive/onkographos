# DäRF

Also called: Monocular Depth Adaptation for boosting Radiance Fields from Sparse Input Views

## Overview
DäRF is a framework for robust few-shot NeRF reconstruction that combines Neural Radiance Fields with Monocular Depth Estimation (MDE) through online complementary training [S1, unofficial]. It solves the ambiguity and inconsistency problems inherent in monocular depth priors by jointly optimizing the NeRF and the MDE network at a specific scene, rather than treating MDE as a static prior [S1, unofficial]. Unlike SCADE [50], which applies pretrained MDE only to seen views and uses probabilistic modeling for uncertainty, DäRF distills the monocular depth prior to NeRF at both seen and unseen views and directly removes ambiguity by finetuning the MDE network with canonical geometry captured by NeRF [S1, unofficial].

## How it works
**Inputs:** A set of input RGB images $S = \{I_i\}$ with ground truth camera poses, and a pretrained MDE network $G_\phi$.
**Outputs:** Optimized NeRF parameters $\Theta$, adapted MDE parameters $\phi$, and a high-quality 3D radiance field.

**Data Structures:**
*   **NeRF Model $F_\Theta$:** A multi-layer perceptron (MLP) mapping 3D coordinates $\mathbf{x} \in \mathbb{R}^3$ and viewing directions $\mathbf{d} \in \mathbb{R}^2$ to color $\mathbf{c}$ and density $\sigma$.
*   **MDE Network $G_\phi$:** A pretrained network (e.g., Mannequin Challenge, DiffDP, DPThybrid) mapping RGB images to relative depth maps $D^*$ [S1, unofficial].
*   **Ray Sampler:** Samples points $\mathbf{x}_t$ along rays $\mathbf{r}$ originating from camera poses.

**Numbered Steps:**

1.  **Depth Prior Extraction:** For each input image $I_i$, the MDE network $G_\phi$ produces a relative depth map $D^*$ [S1, unofficial].
2.  **NeRF Depth Rendering:** The NeRF model $F_\Theta$ renders an absolute depth map $\bar{D}$ for the current camera pose by accumulating sample depths weighted by transmittance and density along rays [S1, unofficial].
3.  **Patch-wise Scale-Shift Alignment:** To overcome the inability of global scale-shift fitting due to poor MDE performance on depth differences between multiple objects, DäRF applies a novel patch-wise scale-shift fitting [S1, unofficial]. For each patch $p$, it computes local scale $w_p$ and shift $q_p$ via least-squares optimization:
    $$ \min_{w_p, q_p} \sum_{k \in p} \| (w_p \cdot \text{sg}(D^*(k)) + q_p) - \bar{D}(k) \|^2 $$
    where $\text{sg}(\cdot)$ is the stop-gradient operator, preventing gradients from flowing into the frozen MDE network during NeRF updates [S1, unofficial].
4.  **Confidence Modeling:** A confidence modeling technique is used to verify accurate depth information, handling ambiguity problems such as relative depths not being multiview-consistent and convexity problems [S1, unofficial]. This generates a confidence mask or weight to filter out unreliable depth predictions.
5.  **Joint Optimization (Complementary Training):** DäRF jointly optimizes NeRF and MDE at a specific scene [S1, unofficial].
    *   **NeRF Update:** The NeRF is updated using the standard volume rendering loss (e.g., MSE between rendered color and ground truth) and the depth consistency loss $L_{depth}$ between the aligned MDE depth and rendered NeRF depth [S1, unofficial].
    *   **MDE Adaptation:** The MDE network $G_\phi$ is finetuned using the canonical geometry captured by NeRF. This adapts the MDE network to produce depths aligned accurately with NeRF geometry, directly removing ambiguity present in the initial MDE output [S1, unofficial].
6.  **Distillation:** The monocular depth prior is distilled to NeRF at both seen and unseen views, enhancing robustness and coherence [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# DäRF Joint Optimization Step
def step_darf(images, poses, nerf_model, mde_model, optimizer_nerf, optimizer_mde):
    # 1. Extract relative depth from MDE
    # G_phi is the MDE network parameters
    relative_depths = mde_model(images)
    
    # 2. Render NeRF outputs
    # F_Theta is the NeRF network parameters
    # ray_sampler generates rays from poses
    rays = ray_sampler(poses)
    rendered_colors, rendered_depths, alphas = nerf_model(rays)
    
    # 3. Patch-wise Scale-Shift Alignment
    # Divide image into patches
    aligned_depths = torch.zeros_like(rendered_depths)
    confidence_masks = torch.ones_like(rendered_depths)
    
    for patch in patches:
        # Extract patch regions
        r_depth_patch = relative_depths[patch]
        n_depth_patch = rendered_depths[patch]
        
        # Stop gradient on MDE depth
        r_depth_patch_sg = r_depth_patch.detach()
        
        # Solve for local scale w and shift q
        # min || w * r_depth_sg + q - n_depth ||^2
        # Closed form solution for least squares
        # w = cov(r, n) / var(r)
        # q = mean(n) - w * mean(r)
        w, q = solve_local_affine(r_depth_patch_sg, n_depth_patch)
        
        # Apply alignment
        aligned_depth_patch = w * r_depth_patch + q
        
        # 4. Confidence Modeling
        # Compute confidence based on consistency or uncertainty
        # [G] Specific confidence formula not detailed in source; 
        # assume a function that returns weights between 0 and 1
        conf = compute_confidence(r_depth_patch, aligned_depth_patch)
        confidence_masks[patch] = conf
        
        aligned_depths[patch] = aligned_depth_patch
        
    # 5. Compute Losses
    # Color Loss
    color_loss = mse(rendered_colors, ground_truth_colors)
    
    # Depth Consistency Loss
    # Weighted by confidence mask
    depth_loss = mse(aligned_depths * confidence_masks, rendered_depths * confidence_masks)
    
    total_loss = color_loss + lambda_depth * depth_loss
    
    # 6. Update NeRF
    optimizer_nerf.zero_grad()
    total_loss.backward(retain_graph=True) # Retain for MDE update if needed
    optimizer_nerf.step()
    
    # 7. MDE Adaptation
    # Adapt MDE to NeRF's canonical geometry
    # [G] The source states MDE is finetuned with canonical geometry.
    # This implies a loss term on MDE output vs NeRF rendered depth.
    # We re-render or use the previously computed aligned depth as target.
    mde_loss = mse(relative_depths, aligned_depths.detach()) # Detach to avoid gradient conflict?
    # Or more likely, the MDE is updated to minimize the difference between its 
    # output and the NeRF's rendered depth after alignment, effectively distilling.
    
    optimizer_mde.zero_grad()
    mde_loss.backward()
    optimizer_mde.step()
```

**Parameters:**
*   $\lambda_{depth}$: Weight for the depth consistency loss. Not given in the sources.
*   Patch size: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Global vs. Local Scale:** Global scale-shift fitting fails when MDE performs poorly on depth differences between multiple objects. Patch-wise fitting is required [S1, unofficial].
*   **Ambiguity:** Monocular depths suffer from ambiguity (e.g., convexity problems, lack of multiview consistency). DäRF handles this via confidence modeling and joint adaptation [S1, unofficial].
*   **Gradient Flow:** The stop-gradient operator $\text{sg}(\cdot)$ is critical to prevent gradients from flowing into the frozen MDE network during the scale-shift alignment step [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher than standard NeRF due to the joint optimization of the MDE network and the patch-wise scale-shift fitting computations. Not given in the sources.
*   **Memory Complexity:** Increased memory usage due to storing MDE network parameters and intermediate depth maps for patch-wise operations. Not given in the sources.
*   **Performance:** Achieves state-of-the-art results quantitatively and qualitatively on real-world indoor and outdoor scene datasets for few-shot NeRF optimization [S1, unofficial].
*   **Strengths:** Robust geometry reconstruction and artifact removal in both unseen and seen viewpoints. Fully exploits MDE by jointly optimizing it with NeRF.
*   **Limitations:** Relies on the quality of the pretrained MDE network. Computational overhead from joint training.
*   **When to use:** Few-shot NeRF reconstruction where standard NeRF fails due to sparse views and ambiguity.
*   **When not to use:** When computational resources are limited and standard NeRF suffices.

## Background
DäRF was proposed to address the weaknesses of methods that use 3D priors extracted from a few input images only, which prevents guidance from encompassing the entire scene [S1, unofficial]. It builds on the strengths of NeRF [31] and pretrained monocular depth estimation (MDE) networks [39, 38, 5]. Unlike SCADE [50], which distills depths by applying pretrained MDE to NeRF at seen views only, DäRF fully exploits the ability of MDE by jointly optimizing NeRF and MDE at a specific scene [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
