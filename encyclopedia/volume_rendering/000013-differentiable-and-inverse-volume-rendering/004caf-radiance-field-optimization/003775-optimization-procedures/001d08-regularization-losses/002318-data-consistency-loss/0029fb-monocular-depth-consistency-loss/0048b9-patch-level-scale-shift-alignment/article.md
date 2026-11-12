# Patch-level Scale-Shift Alignment

Also called: Patch-level fitting, Patch-level scale-shift adjustment, Patch-wise scale-shift fitting, Patch-wise scale-shift invariant loss

## Overview
Patch-level Scale-Shift Alignment is a regularization technique used in differentiable volume rendering to align predicted monocular depth maps with reference or ground truth depths by optimizing scale and shift parameters locally per image patch rather than globally per image [S1, unofficial]. It solves the problem of local depth misalignment errors that arise when a single global affine transformation fails to account for depth inconsistencies among different objects within a scene [S1, unofficial]. Unlike image-level fitting, which computes a single scale and shift for the entire depth map, this method adapts locally, significantly reducing misalignment errors in rendered depth maps [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $S$: The set of images in the scene.
    *   $D^*_i(p)$: The predicted depth value at pixel $p$ for image $i \in S$ (e.g., from a NeRF or MDE prior).
    *   $\bar{D}_i(p)$: The target depth value at pixel $p$ for image $i \in S$ (e.g., from a Monocular Depth Estimation model or ground truth).
    *   $P$: The set of pixels belonging to a specific non-overlapping patch.
*   **Outputs:**
    *   $w_i$: The scale parameter for the patch.
    *   $q_i$: The shift parameter for the patch.
    *   $L$: The total alignment loss.

**Data Structures**
*   **Patch Grid:** The depth map is partitioned into non-overlapping square patches. The standard resolution for these patches is $80 \times 80$ pixels [S1, unofficial]. This size balances local adaptability with computational overhead [S1, unofficial].

**Formulas**
The goal is to find the optimal parameters $h_i = [w_i, q_i]^T$ that minimize the alignment error between the scaled predicted depth and the target depth.

The optimization objective for a single patch is:
$$ (w_i, q_i) = \arg \min_{w_i, q_i} \sum_{p \in P} \| (w_i D^*_i(p) + q_i) - \bar{D}_i(p) \| $$

This can be rewritten as a closed-form least-squares problem. Let $\vec{D}_i(p) = [D^*_i(p), 1]^T$. The problem becomes:
$$ h_{opt} = \arg \min_{h_i} \sum_{p \in P} (\vec{D}_i(p)^T h_i - \bar{D}_i(p))^2 $$

The closed-form solution is:
$$ h_{opt} = \left( \sum_{p \in P} \vec{D}_i(p) \vec{D}_i(p)^T \right)^{-1} \left( \sum_{p \in P} \vec{D}_i(p) \bar{D}_i(p) \right) $$

The total loss $L$ over the scene set $S$ is the sum of errors using these optimal parameters:
$$ L = \sum_{I_i \in S} \sum_{p \in P} \| (w_i D^*_i(p) + q_i) - \bar{D}_i(p) \| $$

**Numbered Steps**
1.  **Partition:** Divide the input depth map into non-overlapping patches of size $80 \times 80$ pixels [S1, unofficial].
2.  **Vector Construction:** For each pixel $p$ in the current patch, construct the feature vector $\vec{D}_i(p) = [D^*_i(p), 1]^T$ and identify the target depth $\bar{D}_i(p)$ [S1, unofficial].
3.  **Accumulate Matrices:** Compute the sum of outer products $\mathbf{A} = \sum_{p \in P} \vec{D}_i(p) \vec{D}_i(p)^T$ and the sum of cross-terms $\mathbf{b} = \sum_{p \in P} \vec{D}_i(p) \bar{D}_i(p)$ for the current patch [S1, unofficial].
4.  **Solve:** Calculate the optimal scale and shift vector $h_{opt} = [w_i, q_i]^T$ by solving the linear system $\mathbf{A} h_{opt} = \mathbf{b}$ (i.e., $h_{opt} = \mathbf{A}^{-1} \mathbf{b}$) [S1, unofficial].
5.  **Compute Loss:** Calculate the residual error $\| (w_i D^*_i(p) + q_i) - \bar{D}_i(p) \|$ for all pixels in the patch and sum them to contribute to the total loss $L$ [S1, unofficial].
6.  **Iterate:** Repeat steps 2-5 for all patches in the image and all images in the batch/scene [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def patch_level_scale_shift_alignment(predicted_depth, target_depth, patch_size=80):
    """
    Computes the patch-level scale-shift alignment loss.
    
    Args:
        predicted_depth (np.ndarray): 2D array of shape (H, W), D*_i(p).
        target_depth (np.ndarray): 2D array of shape (H, W), D_bar_i(p).
        patch_size (int): Size of the square patches (default 80).
        
    Returns:
        float: Total loss L.
        np.ndarray: Array of scale parameters w_i per patch.
        np.ndarray: Array of shift parameters q_i per patch.
    """
    H, W = predicted_depth.shape
    total_loss = 0.0
    scales = []
    shifts = []
    
    # Iterate over non-overlapping patches
    for y in range(0, H, patch_size):
        for x in range(0, W, patch_size):
            # Define patch boundaries
            y_end = min(y + patch_size, H)
            x_end = min(x + patch_size, W)
            
            # Extract patch data
            p_pred = predicted_depth[y:y_end, x:x_end]
            p_target = target_depth[y:y_end, x:x_end]
            
            # Flatten to 1D arrays for vector operations
            pred_flat = p_pred.flatten()
            target_flat = p_target.flatten()
            
            # Construct feature vector D_vec = [D*, 1]
            # Shape: (N, 2)
            D_vec = np.column_stack((pred_flat, np.ones_like(pred_flat)))
            
            # Compute A = sum(D_vec * D_vec^T) -> 2x2 matrix
            # Using dot product of D_vec.T and D_vec
            A = D_vec.T @ D_vec
            
            # Compute b = sum(D_vec * target) -> 2x1 vector
            b = D_vec.T @ target_flat
            
            # Solve for h_opt = [w, q]
            # h_opt = inv(A) @ b
            # Guard against singular matrices (e.g., constant depth patch)
            try:
                h_opt = np.linalg.solve(A, b)
            except np.linalg.LinAlgError:
                # Fallback: if singular, assume identity transform or zero shift/scale adjustment
                # [G] Standard practice is to skip or use pseudo-inverse if singular
                h_opt = np.array([1.0, 0.0])
                
            w_i, q_i = h_opt
            
            # Compute aligned predicted depth
            aligned_pred = w_i * pred_flat + q_i
            
            # Compute loss for this patch: sum of absolute differences (L1 norm of residuals)
            # Note: The source formula uses ||...|| which typically implies L2 in least-squares context,
            # but the explicit sum notation in Eq 1 suggests summing the norms. 
            # Given Eq 2 is least-squares (L2), the loss L in Eq 1 is likely the sum of squared errors 
            # or the source uses ||.|| to denote the residual term in the optimization context.
            # However, Eq 1 explicitly writes ||...|| without square. 
            # Standard "scale-shift invariant loss" often uses L1 or L2. 
            # The optimization (Eq 2) is L2. The loss calculation (Eq 1) is ambiguous.
            # We follow the explicit formula in Eq 1: sum of norms. 
            # If ||.|| is L2 norm of scalar, it's absolute value. If L2 squared, it's square.
            # Given "least-squares criterion" in text, we compute squared error for optimization,
            # but the loss L is often reported as the residual magnitude.
            # To be safe and consistent with "least-squares", we compute the residual sum of squares 
            # for the gradient flow, but the text says "minimize alignment error".
            # Let's stick to the explicit Eq 1: sum of ||residual||. 
            # In 1D, ||x|| is usually |x| (L1) or sqrt(x^2) (L2). 
            # Most implementations of this specific DaRF loss use the residual from the LS fit.
            # We will compute the sum of absolute differences to match the visual "error map" description.
            patch_loss = np.sum(np.abs(aligned_pred - target_flat))
            
            total_loss += patch_loss
            scales.append(w_i)
            shifts.append(q_i)
            
    return total_loss, np.array(scales), np.shifts

# Note: The pseudocode above implements the calculation of parameters and loss.
# In a training loop, the loss is backpropagated. The parameters w_i, q_i are 
# computed via the closed-form solution (non-differentiable w.r.t D* directly in the 
# standard sense unless using implicit differentiation or treating the LS solution 
# as a differentiable layer).
```

**Parameters**
*   **Patch Resolution:** $80 \times 80$ pixels [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Singular Matrices:** If a patch has constant predicted depth (e.g., a uniform background), the matrix $\mathbf{A}$ may be singular. A fallback such as identity scaling ($w=1, q=0$) or pseudo-inverse should be used [G].
*   **Boundary Conditions:** Patches at the image boundaries may be smaller than $80 \times 80$. The algorithm must handle these partial patches correctly by adjusting the summation limits [S1, unofficial].
*   **Depth Inversion:** Monocular depth estimators may output inverse depth or normalized depth. Ensure $D^*$ and $\bar{D}$ are in the same metric space before alignment [G].

## Subtopics

### Image-Level Fitting
Computes a single scale and shift for the entire depth map. Simpler but less accurate for scenes with varying depth scales [S1, unofficial].

### Reliable Depth Estimation via Reprojection
Uses reprojection consistency rather than direct depth alignment [7293-reliable-depth-estimation-via-reprojection.md].

### Pseudo-depth Supervised Loss
Uses pseudo-depth labels for supervision, potentially without explicit scale-shift alignment [7487-pseudo-depth-supervised-loss.md].

### Self-Supervised Depth Losses
Relies on photometric consistency without explicit depth priors [9374-self-supervised-depth-losses.md].

### Depth Prior Loss
General term for losses incorporating depth priors, which may or may not use patch-level alignment [10801-depth-prior-loss.md].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves a single pass over pixels to accumulate sums for the least-squares solution. The matrix inversion is $O(1)$ (2x2 matrix) per patch.
*   **Memory Complexity:** $O(1)$ auxiliary memory per patch for storing the accumulated sums $\mathbf{A}$ and $\mathbf{b}$.
*   **Strengths:** Significantly reduces misalignment errors compared to image-level fitting, especially in scenes with multiple objects at different depths [S1, unofficial].
*   **Limitations:** Computationally more expensive than image-level fitting due to per-patch optimization. Sensitive to patch size selection; too small leads to noise, too large reduces adaptability [S1, unofficial].
*   **When to Use:** When training NeRFs from sparse inputs with monocular depth priors, particularly when global alignment fails to capture local depth variations [S1, unofficial].
*   **When Not to Use:** When computational resources are extremely limited and global alignment suffices, or when depth maps are already well-aligned globally.

## Background
Patch-level Scale-Shift Alignment was introduced in the DaRF (Radiance Fields from Sparse Inputs with Monocular Depth Adaptation) paper [S1, unofficial]. It builds upon the concept of scale-shift invariant loss [39] and least-square alignment [53] used in monocular depth estimation (MDE) prior distillation. The method was developed to address the limitations of image-level fitting, which computes a single set of scale and shift values for an entire depth map, leading to errors in rendered depth maps due to depth inconsistencies among objects [S1, unofficial]. It is applied in DaRF to refine MDE depth maps during sparse-view NeRF training [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
