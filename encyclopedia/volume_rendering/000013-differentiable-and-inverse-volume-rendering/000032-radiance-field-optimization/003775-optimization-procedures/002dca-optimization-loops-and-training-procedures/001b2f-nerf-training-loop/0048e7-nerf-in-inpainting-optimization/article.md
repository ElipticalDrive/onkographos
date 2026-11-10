# NeRF-In Inpainting Optimization

Also called: Free-Form NeRF Inpainting with RGB-D Priors, NeRF object removal, NeRF-In

## Overview
NeRF-In is a framework for removing unwanted objects from a 3D scene represented by a pre-trained Neural Radiance Field (NeRF) without requiring category-specific training data or additional training procedures [S1, unofficial]. It solves the problem of intuitive 3D editing by allowing a user to specify a region for removal via a 2D mask on a single view, then optimizing the NeRF parameters to inpaint that region across all views using RGB-D priors [S1, unofficial]. Unlike methods that require retraining from scratch or category-specific conditional fields, NeRF-In updates an existing model by jointly minimizing color and depth guiding losses derived from 2D image inpainting and depth completion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model $F_\Theta$ with parameters $\Theta$; a user-chosen camera position $o_u$; a rendered view $I_u = F_{\Theta_{image}}(o_u)$; a user-drawn binary mask $M_u$ where $M_u=1$ for pixels outside the masked region (background/valid) and $M_u=0$ inside the region to remove [S1, unofficial].
*   **Output:** Updated NeRF parameters $\tilde{\Theta}$ [S1, unofficial].

**Data Structures:**
*   **NeRF Model $F_\Theta$:** Standard MLP-based radiance field representation (see [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)).
*   **Masks:** Binary masks $M \in \{0, 1\}^{H \times W}$.
*   **Guiding Images:** RGB images $I^G$ and depth images $D^G$ generated via external methods.

**Numbered Steps:**

1.  **Initialization:** Initialize NeRF parameters $\Theta$ (using a pre-trained model for the main method) [S1, unofficial].
2.  **User Masking:** The user draws a free-form mask $M_u$ on a rendered view $I_u$ from the pre-trained NeRF $F_\Theta$, where $M_u=1$ indicates pixels outside the masked region [S1, unofficial].
3.  **Camera Sampling:** Sample $K$ camera positions $O = \{o_s \mid s = 1 \dots K\}$ along a pre-set trajectory (e.g., LLFF test-set trajectory) [S1, unofficial].
4.  **Initial Rendering:** Render RGB images $I_s$ and depth images $D_s$ using $F_\Theta$ for each sampled camera position $o_s \in O$ [S1, unofficial].
5.  **Mask Transfer:** Transfer the user-drawn mask $M_u$ to the sampled views to generate transferred masks $M = \{M_s \mid s = 1 \dots K\}$ using a video object segmentation method (e.g., STCN) [S1, unofficial].
6.  **Guiding Material Generation:**
    *   Generate guiding color images $I^G = \{I_s^G \mid s = 1 \dots K\}$ using an image inpainting method $\rho$: $I_s^G = \rho(I_s, M_s)$ [S1, unofficial].
    *   Generate guiding depth images $D^G = \{D_s^G \mid s = 1 \dots K\}$ using a depth completion method $\tau$: $D_s^G = \tau(D_s, M_s, I_s^G)$ [S1, unofficial].
7.  **View Set Definition:** Define the set of all views $O_{all}$ and outside views $O_{out}$. In the main framework, set $O_{all} = \{o_u\}$ (the user-chosen view) and $O_{out} = O$ (all other sampled views) [S1, unofficial].
8.  **Optimization:** Optimize the NeRF parameters $\Theta$ to obtain $\tilde{\Theta}$ by minimizing the joint inpainting formulation $\Phi(M_u, I_u^G, M, D_s^G)$, which consists of a color-guiding loss $L_{color}(\Theta)$ and a depth-guiding loss $L_{depth}(\Theta)$ [S1, unofficial].

**Formulas:**

The expected color $\hat{C}(r)$ of a camera ray $r(t) = o + td$ is computed via volume rendering integral [S1, unofficial]:

$$
\hat{C}(r) = \sum_{i=1}^{N} T(t_i)(1 - \exp(-\sigma(t_i)\delta_i))c(t_i)
$$

where $T(t_i) = \exp(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j)$, $N$ is the number of quadrature points, $\delta_i = t_{i+1} - t_i$, and $c(t_i), \sigma(t_i)$ are color and density at point $t_i$ produced by $F_\Theta$ [S1, unofficial].

The total color-guiding loss $L_{color}(\Theta)$ is the sum of the all-view color loss $L_{all\_color}(\Theta)$ and the outside-view color loss $L_{out\_color}(\Theta)$ [S1, unofficial]:

$$
L_{color}(\Theta) = L_{all\_color}(\Theta) + L_{out\_color}(\Theta)
$$

where [S1, unofficial]:

$$
L_{all\_color}(\Theta) = \sum_{o \in O_{all}} \| F_{\Theta_{image}}(o) - I_o^G \|_2^2
$$

$$
L_{out\_color}(\Theta) = \sum_{o \in O_{out}} \| (F_{\Theta_{image}}(o) - I_o^G) \odot M_o \|_2^2
$$

Here, $F_{\Theta_{image}}(o)$ is the rendered image from camera position $o$, $I_o^G$ is the guiding image, and $M_o$ is the transferred mask [S1, unofficial]. Note that for $o_u \in O_{all}$, the guiding image is $I_u^G$ [S1, unofficial].

The depth-guiding loss $L_{depth}(\Theta)$ measures the L2 distance between rendered fine/coarse depth images and guiding depth [S1, unofficial]:

$$
L_{depth}(\Theta) = \sum_{o_s \in O} \left( \| D_f(o_s) - D_G(o_s) \|_2^2 + \| D_c(o_s) - D_G(o_s) \|_2^2 \right)
$$

where $D_f(o_s)$ is the fine volume predicted depth image, $D_c(o_s)$ is the coarse volume predicted depth image, and $D_G(o_s)$ is the guiding depth image for camera position $o_s$ [S1, unofficial]. The depth $D_f(o_s)$ is computed by accumulating density $\sigma$ from ray batches during volume rendering [S1, unofficial].

The final optimization objective is [S1, unofficial]:

$$
\tilde{\Theta} := \arg \min_{\Theta} L_{color}(\Theta) + L_{depth}(\Theta)
$$

**Parameters:**
*   $K$: Number of sampled camera positions/views used for inpainting guidance.
*   $M_u$: User-drawn mask on the user-chosen view $I_u$.
*   Batch size: 4,096 [S1, unofficial].
*   Number of sample points in fine model: 128 [S1, unofficial].
*   Number of sample points in coarse model: 64 [S1, unofficial].
*   Optimizer: Adam [S1, unofficial].
*   Training steps for inpainting: 50,000 steps [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_in_inpainting(pretrained_nerf, user_view_cam, user_mask):
    """
    Optimizes a pre-trained NeRF to remove an object specified by a user mask.
    
    Args:
        pretrained_nerf: Pre-trained NeRF model F_Theta.
        user_view_cam: Camera position o_u chosen by the user.
        user_mask: Binary mask M_u (1 for background, 0 for object) on the rendered view I_u.
        
    Returns:
        updated_nerf: NeRF model with updated parameters Theta_tilde.
    """
    # 1. Initialize parameters (already pre-trained)
    theta = pretrained_nerf.parameters
    
    # 2. Sample K camera positions along a pre-set trajectory
    # Example: LLFF test-set trajectory
    K = 10 # Example value, depends on trajectory
    sampled_cams = sample_trajectory(K, user_view_cam)
    
    # 3. Render initial RGB and Depth images for sampled views
    initial_rgbs = []
    initial_depths = []
    for cam in sampled_cams:
        rgb, depth = render_view(pretrained_nerf, cam)
        initial_rgbs.append(rgb)
        initial_depths.append(depth)
        
    # 4. Transfer user mask to sampled views using Video Object Segmentation (e.g., STCN)
    # M_u is on user_view_cam. Transfer to other views.
    transferred_masks = video_object_segmentation(user_mask, user_view_cam, initial_rgbs, sampled_cams)
    
    # 5. Generate Guiding Materials
    # Generate guiding color images using Image Inpainting (e.g., MST inpainting network)
    guiding_rgbs = []
    for i, cam in enumerate(sampled_cams):
        # I_s^G = rho(I_s, M_s)
        guided_rgb = image_inpainting(initial_rgbs[i], transferred_masks[i])
        guiding_rgbs.append(guided_rgb)
        
    # Generate guiding depth images using Depth Completion (e.g., Fast Bilateral Solver)
    guiding_depths = []
    for i, cam in enumerate(sampled_cams):
        # D_s^G = tau(D_s, M_s, I_s^G)
        guided_depth = depth_completion(initial_depths[i], transferred_masks[i], guiding_rgbs[i])
        guiding_depths.append(guidged_depth)
        
    # 6. Define View Sets
    # O_all = {o_u}, O_out = {o_s | s=1...K}
    o_u = user_view_cam
    O_out = sampled_cams
    
    # 7. Optimization Loop
    optimizer = Adam(theta, lr=0.001) # Typical Adam settings
    for step in range(50000):
        optimizer.zero_grad()
        
        # Compute Color Guiding Loss
        # L_all_color: on O_all (only o_u)
        rendered_u = render_view(pretrained_nerf, o_u)[0] # RGB only
        # Note: Guiding image for o_u is generated from user_mask on I_u
        # I_u^G is computed similarly to guiding_rgbs but for the user view
        # Assuming I_u^G is available or computed from user_mask and I_u
        guiding_u = image_inpainting(rendered_u, user_mask)
        loss_all_color = mse(rendered_u, guiding_u)
        
        # L_out_color: on O_out
        loss_out_color = 0.0
        for i, cam in enumerate(O_out):
            rendered_s = render_view(pretrained_nerf, cam)[0]
            guiding_s = guiding_rgbs[i]
            mask_s = transferred_masks[i]
            # (F(o) - I^G) * M
            diff = (rendered_s - guiding_s) * mask_s
            loss_out_color += mse(diff, torch.zeros_like(diff))
            
        loss_color = loss_all_color + loss_out_color
        
        # Compute Depth Guiding Loss
        # L_depth: on O_out (sampled views)
        loss_depth = 0.0
        for i, cam in enumerate(O_out):
            # Render fine and coarse depth
            depth_f, depth_c = render_depth(pretrained_nerf, cam)
            guiding_d = guiding_depths[i]
            
            # L2 distance between rendered and guiding depth
            loss_depth += mse(depth_f, guiding_d) + mse(depth_c, guiding_d)
            
        # Total Loss
        loss = loss_color + loss_depth
        loss.backward()
        optimizer.step()
        
    return NeRFModel(theta)

def render_depth(nerf, cam):
    """
    Renders fine and coarse depth images from a NeRF model.
    Depth is computed by accumulating density sigma from ray batches.
    """
    # Implementation details depend on the specific NeRF architecture's depth rendering
    # Typically involves weighted average of t_i with alpha weights
    # D_f = sum(T_i * (1-alpha_i) * t_i) / sum(T_i * (1-alpha_i))
    # Similar for D_c with different sampling strategy
    pass
```

**Edge Cases and Pitfalls:**
*   Optimizing only with masked MSE (mmse) on the transferred masks without explicit guidance causes the unwanted object to remain in the result [S1, unofficial].
*   Using only depth-guiding loss ($L_{depth}$) introduces color noise outside the masked region and loses color information in flat regions [S1, unofficial].
*   Using multiple inpainted images as color guidance introduces visual inconsistency between views [S1, unofficial].
*   Artifacts in initial guidance materials (from $\rho$ or $\tau$) may propagate to the optimized NeRF [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Optimization requires 50,000 steps using Adam optimizer [S1, unofficial]. On an Intel i7-7800X CPU and GTX-1080 GPU, inpainting takes about five hours per scene [S1, unofficial].
*   **Memory Complexity:** Depends on the NeRF model size and batch size (4,096 rays) [S1, unofficial].
*   **Strengths:** No category-specific training data or procedure required; preserves view consistency; corrects geometry via depth guidance [S1, unofficial].
*   **Limitations:** Relies on the quality of external image inpainting and depth completion methods; may not handle large holes with high variance well if guidance is poor [S1, unofficial].
*   **When to use:** When removing unwanted objects from a pre-trained NeRF without access to category-specific data or the ability to retrain from scratch.
*   **When not to use:** When high-quality category-specific training data is available and a more specialized editing method might yield better results; when the object to remove is very large and guidance methods fail to provide coherent priors.

## Background
NeRF-In was introduced by Hao-Kang Liu, I-Chao Shen, and Bing-Yu Chen in 2022 [S1, unofficial]. It addresses the unintuitive nature of editing pre-trained NeRFs, where parameters are not explicitly associated with scene geometry or appearance [S1, unofficial]. The method builds on prior work in image inpainting [S1, unofficial] and NeRF optimization [S1, unofficial]. It was evaluated on the LLFF dataset and a custom dataset with 24 camera positions per scene [S1, unofficial].

## Sources

[S1] NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors, https://doi.org/10.48550/arxiv.2206.04901, fetched 2026-10-26 (unofficial)
