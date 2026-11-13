# NeRF-In Inpainting Optimization

Also called: Free-Form NeRF Inpainting with RGB-D Priors, NeRF object removal, NeRF-In

## Overview
NeRF-In is a method for removing unwanted objects from a pre-trained Neural Radiance Field (NeRF) by optimizing the model parameters $\Theta$ using color and depth guidance derived from 2D images [S1, unofficial]. It solves the problem of editing 3D scenes without requiring category-specific training data or retraining the entire network from scratch [S1, unofficial]. Unlike sibling methods that may rely on semantic feature distillation [12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md] or diffusion-based pseudo-observations [11860-deceptive-nerf-3dgs-pipeline.md], NeRF-In relies on explicit RGB-D priors generated via standard 2D inpainting and depth completion, optimizing a joint loss to ensure geometric consistency and photometric plausibility across views [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model $F_\Theta$ mapping $(\mathbf{x}, \mathbf{v}) \to (\sigma, \mathbf{c})$; a user-drawn mask $M_u$ on a user-chosen view $I_u$; a trajectory of $K$ camera positions $O = \{o_s \mid s = 1 \dots K\}$ (e.g., from the LLFF test set) [S1, unofficial].
*   **Output:** Updated NeRF parameters $\tilde{\Theta}$ [S1, unofficial].

**Data Structures:**
*   **NeRF Model $F_\Theta$:** Standard MLP-based radiance field. Parameters $\Theta$ include weights and biases for density $\sigma$ and color $\mathbf{c}$ networks [S1, unofficial].
*   **Masks:** Binary masks $M$ where $M=1$ indicates pixels outside the masked region (valid/background) and $M=0$ indicates the masked region (object to remove) [S1, unofficial].
*   **Guidance Images:**
    *   $I^G$: Guiding color images.
    *   $D^G$: Guiding depth images.
    *   These are generated per view $o_s$ using inpainting $\rho$ and depth completion $\tau$ [S1, unofficial].

**Numbered Steps:**

1.  **User Masking:** The user selects a view $o_u$ from the pre-trained NeRF, rendering $I_u = F_{\Theta}^{image}(o_u)$, and draws a free-form mask $M_u$ specifying the unwanted object. $M_u = 1$ for pixels outside the masked region [S1, unofficial].
2.  **View Sampling:** Sample $K$ camera positions $O = \{o_s \mid s = 1 \dots K\}$ along a pre-set trajectory (e.g., LLFF test-set trajectory) [S1, unofficial].
3.  **Mask Transfer:** Transfer the user-drawn mask $M_u$ to the sampled views to generate transferred masks $M = \{M_s \mid s = 1 \dots K\}$ using a video object segmentation method (e.g., STCN) [S1, unofficial].
4.  **Guidance Generation:** For each sampled view $o_s$ with rendered image $I_s$ and depth $D_s$:
    *   Generate guiding color image $I_s^G = \rho(I_s, M_s)$ using a single-image inpainting method $\rho$ (e.g., MST inpainting network) [S1, unofficial].
    *   Generate guiding depth image $D_s^G = \tau(D_s, M_s, I_s^G)$ using a depth completion method $\tau$ (e.g., Fast Bilateral Solver) [S1, unofficial].
5.  **Optimization Setup:** Define the set of all views $O_{all}$ and outside views $O_{out}$. In the main framework, set $O_{all} = \{o_u\}$ (the user-chosen view) and $O_{out} = O$ (all other sampled views) [S1, unofficial].
6.  **Loss Computation:** Compute the total loss $\mathcal{L}(\Theta) = \mathcal{L}_{color}(\Theta) + \mathcal{L}_{depth}(\Theta)$ [S1, unofficial].
    *   **Color-Guiding Loss:** $\mathcal{L}_{color}(\Theta) = \mathcal{L}_{all\_color}(\Theta) + \mathcal{L}_{out\_color}(\Theta)$ [S1, unofficial].
        *   $\mathcal{L}_{all\_color}(\Theta) = \sum_{o \in O_{all}} \| F_{\Theta}^{image}(o) - I_o^G \|_2^2$ [S1, unofficial].
        *   $\mathcal{L}_{out\_color}(\Theta) = \sum_{o \in O_{out}} \| (F_{\Theta}^{image}(o) - I_o^G) \odot M_o \|_2^2$ [S1, unofficial].
    *   **Depth-Guiding Loss:** $\mathcal{L}_{depth}(\Theta) = \sum_{o_s \in O} (\| D_f(o_s) - D_s^G \|_2^2 + \| D_c(o_s) - D_s^G \|_2^2)$ [S1, unofficial].
        *   $D_f(o_s)$ and $D_c(o_s)$ are the fine and coarse volume predicted depth images, computed by accumulating density $\sigma$ from ray batches during volume rendering [S1, unofficial].
7.  **Parameter Update:** Optimize $\Theta$ using the Adam optimizer for 50,000 steps to obtain $\tilde{\Theta}$ [S1, unofficial].

**Formulas:**

The expected color $\hat{C}(\mathbf{r})$ of a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ is computed via volume rendering integral [S1, unofficial]:
$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T(t_i)(1 - \exp(-\sigma(t_i)\delta_i))\mathbf{c}(t_i)
$$
where $T(t_i) = \exp(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j)$, $N$ is the number of quadrature points, $\delta_i = t_{i+1} - t_i$, and $\mathbf{c}(t_i), \sigma(t_i)$ are color and density at point $t_i$ [S1, unofficial].

The total color-guiding loss is [S1, unofficial]:
$$
\mathcal{L}_{color}(\Theta) = \mathcal{L}_{all\_color}(\Theta) + \mathcal{L}_{out\_color}(\Theta)
$$
where $\mathcal{L}_{all\_color}$ measures color difference on the entire user-chosen view, and $\mathcal{L}_{out\_color}$ measures color difference outside the mask on other views [S1, unofficial].

The depth-guiding loss is [S1, unofficial]:
$$
\mathcal{L}_{depth}(\Theta) = \sum_{o_s \in O} (\| D_f(o_s) - D_G(o_s) \|_2^2 + \| D_c(o_s) - D_G(o_s) \|_2^2)
$$
where $D_f, D_c$ are rendered fine/coarse depth images and $D_G$ is the guiding depth [S1, unofficial].

**Parameters:**
*   $K$: Number of sampled camera positions.
*   $M_u$: User-drawn mask ($1$ outside, $0$ inside) [S1, unofficial].
*   Batch size: 4,096 [S1, unofficial].
*   Sample points (fine): 128 [S1, unofficial].
*   Sample points (coarse): 64 [S1, unofficial].
*   Optimizer: Adam [S1, unofficial].
*   Training steps for inpainting: 50,000 [S1, unofficial].

**Pitfalls:**
*   Optimizing only with masked MSE (mmse) on transferred masks without explicit guidance causes the unwanted object to remain [S1, unofficial].
*   Using only depth-guiding loss ($\mathcal{L}_{depth}$) introduces color noise outside the masked region and loses color information in flat regions [S1, unofficial].
*   Using multiple inpainted images as color guidance introduces visual inconsistency between views; the method restricts color guidance to the user-chosen view to maintain consistency [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_in_inpainting(nerf_model, user_view_idx, user_mask, trajectory_indices, 
                       inpainting_fn, depth_completion_fn, optimizer_config):
    """
    Optimizes NeRF parameters to remove an object specified by user_mask on user_view_idx.
    
    Args:
        nerf_model: Pre-trained NeRF model F_Theta.
        user_view_idx: Index of the view where the user draws the mask.
        user_mask: Binary mask M_u (1 outside, 0 inside) for user_view_idx.
        trajectory_indices: List of K camera indices to sample for guidance.
        inpainting_fn: Function rho(I, M) -> I_G.
        depth_completion_fn: Function tau(D, M, I_G) -> D_G.
        optimizer_config: Dict with 'lr', 'steps' (50000), etc.
        
    Returns:
        Updated NeRF model parameters.
    """
    # 1. Setup views
    o_u = trajectory_indices[user_view_idx]
    O = [trajectory_indices[i] for i in trajectory_indices if i != user_view_idx]
    O_all = [o_u]
    O_out = O
    
    # 2. Generate Guidance for User View
    I_u = render_image(nerf_model, o_u)
    I_u_G = inpainting_fn(I_u, user_mask)
    # Depth for user view is not explicitly used in L_depth sum over O, 
    # but needed if user view was in O (it is not in O_out).
    # The paper defines L_depth over O (sampled views excluding user view? 
    # Actually O is defined as sampled positions. O_all = {o_u}, O_out = O.
    # L_depth sums over O. So we need guidance for views in O.
    
    guidance_data = {}
    
    # 3. Generate Guidance for Sampled Views O
    for o_s in O:
        I_s = render_image(nerf_model, o_s)
        D_s = render_depth(nerf_model, o_s) # Coarse/Fine accumulation of sigma
        
        # Transfer mask from user view to o_s using video segmentation (STCN)
        # This is a pre-processing step, assumed to be done or available as M_s
        M_s = transfer_mask(user_mask, o_u, o_s) 
        
        I_s_G = inpainting_fn(I_s, M_s)
        D_s_G = depth_completion_fn(D_s, M_s, I_s_G)
        
        guidance_data[o_s] = {
            'I_G': I_s_G,
            'D_G': D_s_G,
            'M': M_s
        }

    # 4. Optimization Loop
    optimizer = Adam(nerf_model.parameters(), lr=optimizer_config['lr'])
    
    for step in range(optimizer_config['steps']):
        optimizer.zero_grad()
        
        # --- Color Loss ---
        # L_all_color: on O_all (user view)
        I_u_rendered = render_image(nerf_model, o_u)
        # Note: The paper uses I_u_G for the user view. 
        # Does it use mask? Eq 9 sums over O_all. Eq 10 sums over O_out with mask.
        # Eq 9 does not show mask multiplication. 
        # "L_all_color is used to measure the color difference of the entire image"
        loss_all_color = torch.mean((I_u_rendered - I_u_G) ** 2)
        
        # L_out_color: on O_out (other views) with mask
        loss_out_color = 0.0
        for o_s in O_out:
            I_s_rendered = render_image(nerf_model, o_s)
            M_s = guidance_data[o_s]['M']
            I_s_G = guidance_data[o_s]['I_G']
            # Masked MSE: (I - I_G)^2 * M
            # M=1 outside, M=0 inside. We want to match I_G outside.
            # Wait, if M=1 is outside, we want to preserve outside.
            # The loss is || (F - I_G) * M ||^2.
            # If M=1 (outside), we force F to match I_G? 
            # No, I_G is the inpainted image. 
            # For O_out, I_G is generated from I_s and M_s.
            # If M_s=1 (outside), I_G should be close to I_s (original).
            # If M_s=0 (inside), I_G is inpainted.
            # The loss term is || (F - I_G) * M ||^2.
            # This forces F to match I_G ONLY where M=1 (outside).
            # This seems counter-intuitive for "inpainting". 
            # Let's re-read Eq 10: L_out_color = sum || (F(o) - I_o^G) M_o ||^2.
            # And Eq 9: L_all_color = sum || F(o) - I_o^G ||^2.
            # For user view (O_all), we force the whole image to match I_u^G.
            # For other views (O_out), we force the OUTSIDE (M=1) to match I_o^G.
            # This preserves the background consistency.
            mask_term = M_s # 1 outside, 0 inside
            loss_out_color += torch.mean(((I_s_rendered - I_s_G) * mask_term) ** 2)
            
        loss_color = loss_all_color + loss_out_color
        
        # --- Depth Loss ---
        loss_depth = 0.0
        for o_s in O:
            D_f = render_depth_fine(nerf_model, o_s)
            D_c = render_depth_coarse(nerf_model, o_s)
            D_G = guidance_data[o_s]['D_G']
            loss_depth += torch.mean((D_f - D_G) ** 2) + torch.mean((D_c - D_G) ** 2)
            
        total_loss = loss_color + loss_depth
        total_loss.backward()
        optimizer.step()
        
    return nerf_model
```

## Subtopics

### Progressive Inpainting-and-Erasing
Synthesizes new views iteratively and adds them to the training set with conflict detection. Chosen when geometry inconsistencies are severe and iterative refinement is needed. [8658-progressive-inpainting-and-erasing.md]

### Deceptive-NeRF/3DGS Pipeline
Uses diffusion models conditioned on depth and uncertainty to generate synthetic pseudo-observations. Chosen for generative editing where structural priors are weak. [11860-deceptive-nerf-3dgs-pipeline.md]

### DFF (Decomposing NeRF for Editing Via Feature Field Distillation)
Uses semantic feature maps distilled from 2D teacher networks (LSeg, DINO). Chosen for semantically aware editing. [12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md]

### Blended-NeRF
Defines ROI via a 3D bounding box. Chosen for box-based object removal. [12273-blended-nerf.md]

### Adaptive View Selection for Inpainting
Selects optimal views for inpainting guidance. Chosen to minimize computational cost and maximize guidance quality. [12405-adaptive-view-selection-for-inpainting.md]

### NeRF Editing Procedure
General category including inference-time interception (masking/deleting $\sigma$) without retraining. Chosen for fast, non-optimization-based edits. [12491-nerf-editing-procedure.md]

## Operating characteristics
*   **Time Complexity:** Optimization takes approximately 5 hours per scene on an Intel i7-7800X CPU and GTX-1080 GPU for 50,000 steps [S1, unofficial].
*   **Memory Complexity:** Depends on the NeRF model size and batch size (4,096 rays) [S1, unofficial].
*   **Performance:** Generates visually plausible inpainted results on the LLFF dataset and custom datasets with 24 camera positions per scene [S1, unofficial].
*   **Strengths:** No category-specific training data required; handles free-form masks; ensures view consistency via joint optimization [S1, unofficial].
*   **Limitations:** Relies on the quality of 2D inpainting/depth completion priors; artifacts in guidance propagate to 3D; visual inconsistency if multiple views are used as color guidance [S1, unofficial].
*   **When to use:** When removing arbitrary objects from a pre-trained NeRF without access to category-specific data or the ability to retrain from scratch.
*   **When not to use:** When real-time editing is required (optimization is slow); when high-quality 2D inpainting priors are unavailable or unreliable.

## Background
NeRF-In was introduced by Hao-Kang Liu, I-Chao Shen, and Bing-Yu Chen in 2022 [S1, unofficial]. It is the first framework to enable free-form object removal from a pre-trained NeRF without category-specific training data or procedures [S1, unofficial]. It builds on the principles of differentiable volume rendering and 2D image inpainting [S1, unofficial].

## Sources
[S1] NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors, https://doi.org/10.48550/arxiv.2206.04901, fetched 2026-10-26 (unofficial)
