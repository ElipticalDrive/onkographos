# Rendering-based calibration for depth selection

Also called: Rendering-based calibration

## Overview
A post-selection refinement step that evaluates candidate depth changes by rendering held-out calibration views to account for visibility, Gaussian overlap, and alpha-compositing interactions which are not explicitly modeled by the local proxy [S1, unofficial]. It serves as the final quality gate in Adaptive Residual Vector Quantization (ARQ) with Context-Guided Recovery, ensuring that depth adjustments selected via local proxies actually improve the rendered image quality [S1, unofficial].

## How it works
**Inputs and outputs:**
*   **Inputs:**
    *   $I^{\text{Stu,old}}$: The current rendered image using the existing depth and RVQ indices.
    *   $I^{\text{GT}}$: Ground-truth reference images for the calibration views.
    *   $\mathcal{V}_{\text{cal}}$: The set of held-out calibration views.
    *   Candidate groups of pending anchors with proposed depth changes.
    *   $\lambda_R$: Rate-distortion trade-off parameter.
    *   $N_a$: Total number of anchors.
*   **Outputs:**
    *   Accepted or rejected depth changes for each candidate group.
    *   Final depths and RVQ indices for bitstream regeneration.

**Data structures:**
*   **Candidate Groups:** A list of anchor subsets, each containing proposed depth modifications.
*   **Calibration Views:** A set of camera poses and intrinsics used for evaluation.

**Numbered steps:**
1.  **Divide candidates:** Partition the pending anchors into candidate groups to limit computational cost [S1, unofficial].
2.  **Evaluate sequentially:** For each candidate group $b$:
    a.  Apply the candidate depth changes to the current state.
    b.  Render the scene from all views in $\mathcal{V}_{\text{cal}}$ using the modified depths to obtain $I_b^{\text{Stu,new}}$ [S1, unofficial].
    c.  Calculate the average rendering distortion $\mathcal{D}_{\text{cal}}$ using the formula below.
    d.  Compute the induced change in distortion $\Delta \mathcal{D}_{\text{cal},b}$ relative to the previous state $I^{\text{Stu,old}}$ [S1, unofficial].
    e.  Calculate the change in code length $\Delta \widetilde{R}_b^{\text{CDF}}$ from integer CDFs after applying the candidate group [S1, unofficial].
    f.  Compute the rendering-based rate-distortion cost $\Delta \mathcal{J}_b^{\text{cal}}$ [S1, unofficial].
    g.  If $\Delta \mathcal{J}_b^{\text{cal}} \le 0$, accept the candidate update; otherwise, retain the previous state [S1, unofficial].
    h.  Update $I^{\text{Stu,old}}$ to $I_b^{\text{Stu,new}}$ if accepted, or keep it unchanged if rejected.
3.  **Regenerate bitstream:** After all candidate groups are evaluated, regenerate the arithmetic-coded bitstream using the final accepted depths and RVQ indices [S1, unofficial].

**Formulas:**
*   **Rendering Loss:**
    $$ \mathcal{L}_{\text{render}}(I, I^{\text{ref}}) = (1 - \gamma_{\text{SSIM}}) \text{MAE}(I, I^{\text{ref}}) + \gamma_{\text{SSIM}} [1 - \text{SSIM}(I, I^{\text{ref}})] $$
    where $I^{\text{ref}} \in \{I^{\text{GT}}, I^{\text{Tea}}\}$ denotes either the ground-truth image or the teaching image [S1, unofficial]. $\gamma_{\text{SSIM}}$ is a weighting factor.

*   **Average Rendering Distortion:**
    $$ \mathcal{D}_{\text{cal}} \left( I^{\text{Stu}}, I^{\text{GT}} \right) = \frac{1}{|\mathcal{V}_{\text{cal}}|} \sum_{v \in \mathcal{V}_{\text{cal}}} \mathcal{L}_{\text{render}} \left( I_v^{\text{Stu}}, I_v^{\text{GT}} \right) $$
    where $\mathcal{V}_{\text{cal}}$ is the set of calibration views [S1, unofficial].

*   **Distortion Change:**
    $$ \Delta \mathcal{D}_{\text{cal},b} = \mathcal{D}_{\text{cal}} \left( I_b^{\text{Stu,new}}, I^{\text{GT}} \right) - \mathcal{D}_{\text{cal}} \left( I^{\text{Stu,old}}, I^{\text{GT}} \right) $$
    where $I_b^{\text{Stu,new}}$ is the rendering with the candidate applied and $I^{\text{Stu,old}}$ is the previous rendering [S1, unofficial].

*   **Rate-Distortion Cost:**
    $$ \Delta \mathcal{J}_b^{\text{cal}} = \Delta \mathcal{D}_{\text{cal},b} + \lambda_R \frac{\Delta \widetilde{R}_b^{\text{CDF}}}{N_a} $$
    where $\Delta \widetilde{R}_b^{\text{CDF}}$ is the change in code length from integer CDFs after applying candidate group $b$, $N_a$ is the number of anchors, and $\lambda_R$ controls the rate-distortion trade-off [S1, unofficial].

**Parameters:**
*   $\lambda_R$: Rate-distortion trade-off parameter during calibration. Typical value: Not given in the sources.
*   $\gamma_{\text{SSIM}}$: Weighting factor for SSIM in the rendering loss. Typical value: Not given in the sources.

**Edge cases and pitfalls:**
*   **Computational Cost:** Rendering calibration views for every candidate group is expensive. Grouping candidates is necessary to limit cost [S1, unofficial].
*   **Visibility and Overlap:** The local proxy does not model visibility, Gaussian overlap, or alpha-compositing interactions. This step is critical because these effects are only captured by full rendering [S1, unofficial].
*   **Bitstream Regeneration:** The bitstream must be regenerated after calibration, not just updated incrementally, to ensure consistency with the final depths and indices [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rendering_based_calibration(candidates, calibration_views, current_depths, current_rvq_indices, 
                                ground_truth_images, lambda_R, N_a, gamma_SSIM):
    """
    Evaluates candidate depth changes using rendering-based calibration.
    
    Args:
        candidates: List of candidate groups, each containing anchor indices and proposed depth changes.
        calibration_views: Set of camera poses for calibration.
        current_depths: Current depth values for all anchors.
        current_rvq_indices: Current RVQ indices for all anchors.
        ground_truth_images: Dictionary mapping view ID to ground truth image.
        lambda_R: Rate-distortion trade-off parameter.
        N_a: Total number of anchors.
        gamma_SSIM: Weighting factor for SSIM.
        
    Returns:
        final_depths: Updated depth values after calibration.
        final_rvq_indices: Updated RVQ indices (unchanged in this step, but used for bitstream).
    """
    
    # Helper function for rendering loss
    def rendering_loss(I_pred, I_ref):
        mae = mean_absolute_error(I_pred, I_ref)
        ssim = structural_similarity(I_pred, I_ref)
        return (1 - gamma_SSIM) * mae + gamma_SSIM * (1 - ssim)
    
    # Helper function for average rendering distortion
    def calc_distortion(depths, rvq_indices, views, gt_images):
        total_distortion = 0.0
        for view in views:
            # Render image using current depths and RVQ indices for this view
            I_stu = render_gaussian_splatting(depths, rvq_indices, view)
            I_gt = gt_images[view.id]
            total_distortion += rendering_loss(I_stu, I_gt)
        return total_distortion / len(views)
    
    # Helper function for code length change
    def calc_code_length_change(candidate_group, current_depths, current_rvq_indices):
        # Calculate the change in code length from integer CDFs
        # This depends on the specific arithmetic coding implementation
        # For simplicity, assume a function that computes delta R based on depth changes
        return compute_delta_r_cdf(candidate_group, current_depths)
    
    # Initial state
    current_depths = current_depths.copy()
    current_rvq_indices = current_rvq_indices.copy()
    
    # Evaluate each candidate group sequentially
    for group in candidates:
        # Apply candidate changes temporarily
        temp_depths = current_depths.copy()
        for anchor_idx, new_depth in group.proposed_changes:
            temp_depths[anchor_idx] = new_depth
        
        # Render and calculate distortion with new depths
        D_new = calc_distortion(temp_depths, current_rvq_indices, calibration_views, ground_truth_images)
        D_old = calc_distortion(current_depths, current_rvq_indices, calibration_views, ground_truth_images)
        
        # Calculate distortion change
        delta_D = D_new - D_old
        
        # Calculate rate change
        delta_R = calc_code_length_change(group, current_depths, current_rvq_indices)
        
        # Calculate rate-distortion cost
        delta_J = delta_D + lambda_R * (delta_R / N_a)
        
        # Accept or reject
        if delta_J <= 0:
            current_depths = temp_depths
            # Note: RVQ indices are not changed in this step, only depths
        # Else: retain previous state (current_depths remains unchanged)
    
    # Regenerate bitstream with final depths and RVQ indices
    # This step is conceptual; actual bitstream generation is handled by the encoder
    final_depths = current_depths
    final_rvq_indices = current_rvq_indices
    
    return final_depths, final_rvq_indices

def mean_absolute_error(I_pred, I_ref):
    return np.mean(np.abs(I_pred - I_ref))

def structural_similarity(I_pred, I_ref):
    # Placeholder for SSIM calculation
    # In practice, use a library or implement SSIM
    return 1.0 # Placeholder
```

## Operating characteristics
*   **Time Complexity:** Proportional to the number of candidate groups and the number of calibration views. Each evaluation requires rendering the scene from all calibration views, which is computationally expensive [S1, unofficial].
*   **Memory Complexity:** Requires storing ground-truth images for calibration views and intermediate rendered images. Memory usage scales with the number of calibration views and image resolution [S1, unofficial].
*   **Strengths:** Accounts for complex interactions like visibility and alpha-compositing that local proxies miss, leading to higher rendering quality [S1, unofficial].
*   **Limitations:** High computational cost due to repeated rendering. Requires a set of held-out calibration views [S1, unofficial].
*   **When to use:** When high rendering quality is critical and computational resources allow for post-selection refinement [S1, unofficial].
*   **When not to use:** When real-time or low-latency compression is required, or when computational resources are severely limited [S1, unofficial].

## Background
*   **Origin:** Introduced in "3D Gaussian Splatting Data Compression via Adaptive Residual Quantization and Error Compensation" [S1, unofficial].
*   **History:** Part of the ARQ method for compressing 3D Gaussian Splatting data, addressing the limitations of local proxy-based depth selection by incorporating full rendering feedback [S1, unofficial].
*   **Usage:** Used in the context of 3D Gaussian Splatting compression to refine depth selections and improve the visual quality of the compressed representation [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Data Compression via Adaptive Residual Quantization and Error Compensation, https://doi.org/10.21203/rs.3.rs-10881546/v1, fetched 2026-10-26 (unofficial)
