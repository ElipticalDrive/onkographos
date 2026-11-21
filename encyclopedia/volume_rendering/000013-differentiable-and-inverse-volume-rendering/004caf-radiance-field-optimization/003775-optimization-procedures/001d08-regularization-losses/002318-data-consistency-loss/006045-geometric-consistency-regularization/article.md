# Geometric Consistency Regularization

Also called: Consistency-aware Fine-tuning Scheme

## Overview
Geometric Consistency Regularization enforces multi-view geometric constraints during radiance-field optimization to resolve ambiguities in sparse-view reconstruction. It distinguishes itself from sibling losses by filtering or weighting correspondences based on their mutual geometric agreement, thereby eliminating distorted geometries and artifacts common in unconstrained optimization [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images with known camera poses; a current radiance field or Gaussian splatting model; a feature matching module providing candidate correspondences between views.
*   **Outputs:** A regularized loss term added to the training objective; refined geometry (surface or density field).

**Data Structures:**
*   **Correspondences:** A set of 2D pixel coordinates $(u, v)$ in reference and target views, mapped to 3D points via depth or ray intersection.
*   **Threshold:** A scalar parameter $\tau_{ray}$ defining the maximum allowable geometric deviation for a correspondence to be considered valid [S2].

**Steps:**
1.  **Feature Matching:** Extract features from input views and identify candidate 2D correspondences between a reference view and target views [S2].
2.  **Geometric Verification:** For each correspondence, compute the geometric consistency error. This typically involves projecting the 3D point implied by the reference view into the target view and measuring the reprojection error or distance to the target ray [S2].
3.  **Filtering:** Compare the geometric error against the threshold $\tau_{ray}$.
    *   If error $< \tau_{ray}$, the correspondence is marked as **valid**.
    *   If error $\ge \tau_{ray}$, the correspondence is marked as **invalid** and excluded from the regularization loss [S2].
4.  **Loss Computation:** Compute the regularization loss only over the set of valid correspondences. This loss penalizes deviations from the geometrically consistent 3D structure [S2].
5.  **Hybrid Supervision (Optional):** In regions lacking valid geometric correspondences, apply standard photometric supervision (e.g., FreeNeRF procedure) to a randomized selection of rays to ensure full image coverage [S2].

**Parameters:**
*   $\tau_{ray}$: Geometry regularization threshold.
    *   LLFF dataset: $\tau_{ray} = 0.15$ [S2].
    *   DTU dataset: $\tau_{ray} = 0.0017$ [S2].
*   $n_t, n_{ref}$: Number of target/reference rays for photometric backup. Set to 50 per iteration to balance ratio [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_geometric_consistency_loss(
    reference_view, 
    target_views, 
    model, 
    correspondences, 
    tau_ray
):
    """
    Computes geometric consistency loss by filtering correspondences.
    
    Args:
        reference_view: Reference image and camera pose.
        target_views: List of target images and camera poses.
        model: Current radiance field or Gaussian model.
        correspondences: List of (ref_uv, target_uv) pairs from feature matching.
        tau_ray: Threshold for geometric consistency.
        
    Returns:
        loss: Scalar geometric consistency loss.
    """
    valid_correspondences = []
    
    for ref_uv, target_uv in correspondences:
        # 1. Get 3D point from reference view
        # [G] Assuming model can sample depth or ray origin/direction
        ref_ray = get_ray_from_uv(reference_view.camera, ref_uv)
        ref_depth = model.sample_depth(ref_ray)
        ref_point_3d = get_world_point(ref_ray, ref_depth)
        
        # 2. Project to target view
        target_uv_projected = project_to_uv(target_views.camera, ref_point_3d)
        
        # 3. Compute geometric error
        # [G] Euclidean distance in 2D image space or ray distance
        error = distance(ref_uv, target_uv_projected)
        
        # 4. Filter based on threshold
        if error < tau_ray:
            valid_correspondences.append((ref_uv, target_uv))
            
    # 5. Compute loss on valid correspondences
    # [G] Specific loss function (e.g., L2 distance between rendered and target features/depth)
    loss = 0.0
    for ref_uv, target_uv in valid_correspondences:
        ref_rendered = model.render(reference_view.camera, ref_uv)
        target_rendered = model.render(target_views.camera, target_uv)
        loss += loss_function(ref_rendered, target_rendered)
        
    return loss / max(len(valid_correspondences), 1)
```

## Subtopics

### Geometry Regularization Threshold
A threshold parameter $\tau_{ray}$ for the geometry regularization module that filters correspondences based on geometric consistency. It balances matching accuracy against geometric detail preservation.
*   **Parameter:** LLFF dataset: $\tau_{ray} = 0.15$. DTU dataset: $\tau_{ray} = 0.0017$ [S2].
*   **Pitfall:** Higher $\tau_{ray}$ values introduce inaccurate correspondences, while lower values produce less detailed geometry descriptions. Both extremes negatively impact PSNR [S2].
*   **Performance:** LLFF ablation ($\tau_{ray}$ 0.05 to 0.35, step 0.05): Optimal PSNR 20.33 at 0.15. SSIM/LPIPS remain stable. DTU ablation ($\tau_{ray}$ 0.0005 to 0.002, step ~0.0002): Optimal PSNR 18.71 at 0.0017 [S2].

## Operating characteristics
*   **Time Complexity:** Adds overhead for feature matching and geometric verification per iteration. The number of training iterations is specified as 44k for DTU (3 views) and 70k for LLFF [S2].
*   **Memory Complexity:** Requires storing correspondence lists and intermediate 3D points.
*   **Performance:** On the DTU dataset, adding consistency-aware fine-tuning reduces Chamfer distance from 1.28 (Pixel+Patch) to 1.21 [S1].
*   **Strengths:** Produces noticeably cleaner results and eliminates distorted geometries compared to schemes without it [S1]. Avoids sub-optimal solutions caused by radiance ambiguity [S1].
*   **Limitations:** Relies heavily on the quality of initial feature matching; sparse correspondences may leave regions unsupervised without hybrid photometric backup [S2].

## Background
Originated in the context of few-shot neural rendering and sparse-view surface reconstruction. SGCNeRF introduced sparse geometric consistency guidance to leverage sparse geometric constraints [S2]. SparseNeuS applied a consistency-aware fine-tuning scheme to improve generalizable neural surface reconstruction from sparse views [S1].

## Sources
[S1] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
[S2] SGCNeRF: Few-Shot Neural Rendering via Sparse Geometric Consistency Guidance, https://doi.org/10.1109/tcsvt.2025.3610950, fetched 2026-10-30
