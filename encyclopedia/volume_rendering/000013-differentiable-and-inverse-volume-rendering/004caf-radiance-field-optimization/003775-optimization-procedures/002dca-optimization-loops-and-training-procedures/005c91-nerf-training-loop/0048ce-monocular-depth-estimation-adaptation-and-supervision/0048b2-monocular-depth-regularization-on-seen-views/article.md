# Monocular Depth Regularization on Seen Views

Also called: Patch-wise depth alignment, Seen-view depth distillation

## Overview
This method aligns pseudo-depth maps from a pre-trained monocular depth estimation (MDE) network with NeRF-rendered depth for seen input views to stabilize geometry optimization [S1, unofficial]. It differs from global scale-fitting approaches by computing local scale and shift parameters per image patch, preventing dominant objects from biasing the depth alignment and thereby preserving local geometric details [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of seen view images $S$, a pre-trained MDE network $G_\phi$, and a NeRF model $F_\Theta$.
*   **Output:** A depth consistency loss term $L_{seen}$ added to the training objective.

**Data Structures:**
*   **Patches:** The image is divided into non-overlapping or overlapping patches $P$. The specific patch size and stride are not given in the sources [G].
*   **Depth Maps:**
    *   $D^*_i$: Relative depth map from MDE for image $I_i$.
    *   $\bar{D}_i$: Absolute depth map rendered by NeRF for view $i$.

**Parameters:**
*   $w_i, q_i$: Local scale and shift parameters for a specific patch.
*   $\text{sg}(\cdot)$: Stop-gradient operator.

**Steps:**
1.  Iterate through each seen view image $I_i \in S$.
2.  Render the absolute depth map $\bar{D}_i$ from the NeRF $F_\Theta$ for the camera pose of $I_i$.
3.  Compute the relative pseudo-depth map $D^*_i = G_\phi(I_i)$ using the pre-trained MDE network.
4.  For each patch $P$ in the image:
    a. Identify the set of pixels $p \in P$.
    b. Compute the optimal scale $w_i$ and shift $q_i$ by minimizing the least squares error between the scaled MDE depth and the NeRF depth:
       $$ \min_{w_i, q_i} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \|^2 $$
    c. Calculate the per-patch loss contribution using the aligned depths.
5.  Sum the losses over all patches and all seen views.

**Formula:**
The total depth consistency loss for seen views is defined as [S1, unofficial]:
$$ L_{seen} = \sum_{I_i \in S} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \| $$
where $\| \cdot \|$ denotes the L1 or L2 norm (not specified, L1 is common in such contexts but not explicitly stated) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_seen_depth_loss(views, nerf_model, mde_model, patch_size):
    total_loss = 0.0
    for view in views:
        # 1. Render NeRF depth
        # Assumes nerf_model.render_depth returns a tensor of shape (H, W)
        nerf_depth = nerf_model.render_depth(view.camera_pose)
        
        # 2. Get MDE depth
        # Assumes mde_model returns relative depth tensor of shape (H, W)
        mde_depth = mde_model(view.image)
        
        # 3. Iterate over patches
        # Implementation detail: H, W assumed divisible by patch_size or handled via padding/cropping
        H, W = nerf_depth.shape
        for y in range(0, H, patch_size):
            for x in range(0, W, patch_size):
                # Define patch boundaries
                y_end = min(y + patch_size, H)
                x_end = min(x + patch_size, W)
                
                # Extract patch tensors
                patch_mde = mde_depth[y:y_end, x:x_end]
                patch_nerf = nerf_depth[y:y_end, x:x_end]
                
                # 4. Compute scale and shift via least squares
                # Solve min_w,q sum( (w * mde + q - nerf)^2 )
                # Let m = patch_mde, n = patch_nerf
                # w = (Cov(m,n)) / Var(m) if Var(m) > 0, else 1.0
                # q = mean(n) - w * mean(m)
                
                m_mean = patch_mde.mean()
                n_mean = patch_nerf.mean()
                
                # Centered variables
                m_c = patch_mde - m_mean
                n_c = patch_nerf - n_mean
                
                var_m = (m_c ** 2).sum()
                cov_mn = (m_c * n_c).sum()
                
                if var_m > 1e-6:
                    w = cov_mn / var_m
                    q = n_mean - w * m_mean
                else:
                    w = 1.0
                    q = 0.0
                
                # 5. Calculate loss with stop-gradient on MDE depth
                # Note: In a real autograd framework, w and q might need to be detached 
                # or computed via a differentiable solver if gradients are needed through them.
                # The source specifies sg(D*), implying w and q are computed on detached values 
                # or the loss is only backpropagated to NeRF.
                aligned_mde = w * patch_mde.detach() + q
                
                patch_loss = (aligned_mde - patch_nerf).abs().sum() # Assuming L1
                total_loss += patch_loss
                
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Dominant Object Bias:** Global scale/shift fitting favors large or high-contrast objects, leading to ill-fitted depths in less dominant sections [S1, unofficial]. Patch-wise adjustment alleviates this.
*   **Computational Bottleneck:** Rendering full-resolution depth maps for every patch can be expensive. The patch-wise approach helps mitigate this by allowing smaller, localized computations, though full image rendering is still required for the NeRF depth [S1, unofficial].
*   **Gradient Flow:** The stop-gradient operator $\text{sg}(\cdot)$ is critical to prevent gradients from flowing into the pre-trained MDE network $G_\phi$, which is frozen [S1, unofficial].

## Subtopics

### [Transient-Aware Depth Prior](13096-transient-aware-depth-prior.md)
Incorporates geometric constraints from MDE but filters supervision using transient confidence weights to enforce consistency only on static geometry.

## Operating characteristics
*   **Time Complexity:** Depends on the number of seen views, the number of patches, and the cost of MDE inference and NeRF depth rendering. The patch-wise least squares solution is $O(N)$ per patch where $N$ is the number of pixels in the patch.
*   **Memory Complexity:** Requires storing depth maps for the current batch of views. Patch-wise processing allows for lower peak memory usage compared to full-image alignment if processed sequentially.
*   **Strengths:** Preserves local geometric details by avoiding global bias; stabilizes training for sparse inputs.
*   **Limitations:** Relies on the quality of the pre-trained MDE network; patch boundaries may introduce discontinuities if not handled carefully.
*   **When to use:** When training NeRFs on sparse or unconstrained image collections where global depth alignment fails due to varying object scales or dominant foregrounds.

## Background
The method was introduced in the DaRF paper, which proposes boosting radiance fields from sparse inputs using monocular depth adaptation [S1, unofficial]. It addresses the issue that global scale fitting undermines local statistics, a problem highlighted in prior work [62] cited by the authors.

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
