# Monocular Depth Regularization on Seen Views

Also called: Patch-wise depth alignment, Seen-view depth distillation

## Overview
This method applies a depth consistency loss between rendered NeRF depth and pseudo-depth maps from a pre-trained Monocular Depth Estimation (MDE) network for input views that are present in the training set [S1, unofficial]. It differs from global scale/shift fitting by computing alignment parameters per image patch, which prevents dominant objects from skewing the alignment and preserves local geometric details [S1, unofficial]. This approach also mitigates the computational cost associated with full-image rendering by operating on localized regions [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $S$: The set of seen (training) view images.
    *   $G_\phi$: A pre-trained MDE network with parameters $\phi$ [S1, unofficial].
    *   $\mathcal{N}$: The NeRF model capable of rendering depth $\bar{D}$ [S1, unofficial].
    *   $P$: A set of pixels defining a spatial patch within an image [S1, unofficial].
*   **Outputs:**
    *   $L_{\text{seen}}$: The depth consistency loss value for the current batch/iteration [S1, unofficial].

**Data Structures**
*   **Pseudo-depth map $D^*$:** A 2D array of relative depth values predicted by $G_\phi$.
*   **Rendered depth $\bar{D}$:** A 2D array of absolute depth values computed by the NeRF, where $\bar{D}(r) = \sum_i w_i y_i$ [Parent].
*   **Alignment parameters $(w_i, q_i)$:** Two scalar values per patch, representing scale and shift [S1, unofficial].

**Numbered Steps**
1.  Iterate over each seen view image $I_i \in S$.
2.  Render the NeRF depth map $\bar{D}_i$ for the camera pose corresponding to $I_i$ [S1, unofficial].
3.  Extract the pseudo-depth map $D^*_i$ by passing $I_i$ through the frozen MDE network: $D^*_i = G_\phi(I_i)$ [S1, unofficial].
4.  For each patch $P$ in the image:
    a.  Identify the subset of pixels in $P$.
    b.  Compute the scale $w_i$ and shift $q_i$ that minimize the least-squares error between the aligned pseudo-depth and the rendered depth within the patch:
        $$ \min_{w_i, q_i} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \|^2 $$
        where $\text{sg}(\cdot)$ is the stop-gradient operator [S1, unofficial].
    c.  Calculate the per-pixel loss contribution using the computed $w_i$ and $q_i$.
5.  Sum the losses over all patches $P$ and all seen views $I_i \in S$ to obtain $L_{\text{seen}}$ [S1, unofficial].

**Formula**
The total loss is defined as [S1, unofficial]:
$$ L_{\text{seen}} = \sum_{I_i \in S} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \| $$

**Parameters**
*   $w_i, q_i$: Scale and shift parameters for the current patch, computed via least squares [S1, unofficial].
*   $\text{sg}(\cdot)$: Stop-gradient operation applied to the monocular depth $D^*_i$ to prevent gradient flow into the MDE network [S1, unofficial].
*   Typical values: Not given in the sources.

**Edge Cases and Pitfalls**
*   **Global Fitting Bias:** Using global statistics for scale/shift fitting favors dominant objects, leading to ill-fitted depths in less dominant sections due to inconsistencies in predicted depth differences [S1, unofficial]. Patch-wise adjustment alleviates this [S1, unofficial].
*   **Computational Bottleneck:** The patch-based approach helps overcome the computational bottleneck of full image rendering [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_seen_depth_loss(nerf_model, mde_model, seen_views, patches):
    """
    Computes the monocular depth regularization loss on seen views using patch-wise alignment.
    
    Args:
        nerf_model: The NeRF model instance with a render_depth method.
        mde_model: The pre-trained MDE model instance with a forward method.
        seen_views: List of image tensors or paths for training views.
        patches: List of pixel coordinate sets or masks defining patches for each view.
        
    Returns:
        total_loss: Scalar tensor representing L_seen.
    """
    total_loss = 0.0
    
    for view_idx, image in enumerate(seen_views):
        # Step 2: Render NeRF depth for the current view
        # Assumes nerf_model.render_depth(camera_pose) returns a 2D tensor of depth values
        rendered_depth = nerf_model.render_depth(camera_pose=view_idx)
        
        # Step 3: Extract pseudo-depth from pre-trained MDE network
        # G_phi(I_i)
        pseudo_depth = mde_model(image)
        
        # Apply stop-gradient to pseudo_depth to prevent gradients flowing into MDE
        pseudo_depth = pseudo_depth.detach()
        
        # Step 4: Process each patch
        for patch_pixels in patches[view_idx]:
            # Extract depth values for the current patch
            p_rendered = rendered_depth[patch_pixels]
            p_pseudo = pseudo_depth[patch_pixels]
            
            # Step 4b: Compute scale (w) and shift (q) via least squares
            # Solve min_{w,q} sum( (w * p_pseudo + q) - p_rendered )^2
            # This is a linear regression problem: y = w*x + q
            # Using numpy/polyfit or direct least squares solution
            # x = p_pseudo, y = p_rendered
            
            # Handle potential numerical instability if variance is near zero
            if torch.var(p_pseudo) < 1e-6:
                w = 1.0
                q = torch.mean(p_rendered) - torch.mean(p_pseudo)
            else:
                # Least squares solution for w and q
                # w = cov(x,y) / var(x)
                # q = mean(y) - w * mean(x)
                w = torch.cov(p_pseudo.flatten(), p_rendered.flatten())[0, 1] / torch.var(p_pseudo)
                q = torch.mean(p_rendered) - w * torch.mean(p_pseudo)
            
            # Step 4c: Calculate loss for this patch
            # L_patch = sum( || (w * sg(p_pseudo) + q) - p_rendered || )
            # Note: sg is already applied via detach() above
            aligned_pseudo = w * p_pseudo + q
            patch_loss = torch.norm(aligned_pseudo - p_rendered, p=1) # L1 norm as per formula ||...||
            
            total_loss += patch_loss
            
    return total_loss
```

## Subtopics

### Monocular Depth Regularization on Unseen Views
Extends the supervision to novel viewpoints by rendering depth from the NeRF and comparing it with MDE predictions on the rendered color image. [Monocular Depth Regularization on Unseen Views](12354-monocular-depth-regularization-on-unseen-views.md)

### NeRF-guided MDE Adaptation
Adapts the MDE network itself using NeRF predictions, rather than just using MDE to supervise NeRF. [NeRF-guided MDE Adaptation](12333-nerf-guided-mde-adaptation.md)

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of seen views and the number of patches. The least-squares computation is $O(|P|)$ per patch.
*   **Memory Complexity**: Requires storing depth maps for the current batch and patches.
*   **Strengths**: Mitigates the bias of global scale/shift fitting towards dominant objects; reduces computational cost compared to full-image global fitting; improves local geometric consistency [S1, unofficial].
*   **Limitations**: Relies on the quality of the pre-trained MDE network; patch selection strategy is not specified in the source [S1, unofficial].
*   **When to use**: When training NeRFs with sparse input views and needing to enforce geometric consistency using monocular priors, especially when local details are important [S1, unofficial].

## Background
The method was introduced in the DaRF paper (2023) to boost radiance fields from sparse inputs using monocular depth adaptation [S1, unofficial]. It addresses the issue where global scale/shift fitting undermines local statistics in NeRF training [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
