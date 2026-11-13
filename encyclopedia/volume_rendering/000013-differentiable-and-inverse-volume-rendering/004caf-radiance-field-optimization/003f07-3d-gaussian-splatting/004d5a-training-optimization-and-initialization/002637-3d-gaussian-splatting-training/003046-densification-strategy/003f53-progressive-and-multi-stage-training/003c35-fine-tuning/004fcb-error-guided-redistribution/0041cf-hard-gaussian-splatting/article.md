# Hard Gaussian Splatting

Also called: REHGS (Rendering Error guided HGS), Rendering Error guided HGS, Rendering Error guided Hard Gaussian Splatting

## Overview
Hard Gaussian Splatting (HGS) is a training optimization method that addresses artifacts in 3D Gaussian Splatting (3DGS) by identifying and growing "hard Gaussians" based on multi-view significant positional gradients and local rendering errors [S1]. Unlike standard 3DGS, which relies on averaged view-space positional gradients that smooth out local discrepancies, HGS isolates Gaussians causing blurring or needle-like artifacts to improve Novel View Synthesis (NVS) fidelity [S1]. It builds on [Error-Guided Redistribution](14037-error-guided-redistribution.md) by explicitly linking Gaussian size and gradient magnitude to pixel-level reconstruction quality.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$, a set of training viewpoints $J$, ground truth images $\{I_j\}$, and rendered images $\{\hat{I}_j\}$.
*   **Output:** A subset of "hard Gaussians" identified for densification/growth during the training interval $M$.

**Data Structures:**
*   **3D Gaussian $G_i$:** Parameterized by mean vector $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, color $c_i \in \mathbb{R}^3$, and opacity $\alpha_i \in \mathbb{R}$ [S1].
*   **View-space Positional Gradient $g_{i,j}$:** The gradient of the rendering loss with respect to the position $\mu_i$ of Gaussian $G_i$ under viewpoint $j$ [S1].
*   **Contribution Set $S_i$:** The set of pixels $u$ in the image plane $P$ where Gaussian $G_i$ has the maximum rendering contribution [S1].

**Rendering Model:**
A 3D Gaussian $G_i$ is splatted to a 2D Gaussian $G_{i,2D}$. The color $c(u)$ of a pixel $u$ is computed via alpha-blending [S1]:
$$
c(u) = \sum_{i=1}^{N} w_i c_i, \quad w_i = \alpha_i G_{i,2D}(u) \prod_{k=1}^{i-1} (1 - \alpha_k G_{k,2D}(u))
$$
where $N$ is the number of Gaussians sorted by depth [S1].

**Method Steps:**
HGS identifies hard Gaussians using two complementary strategies: Positional Gradient Driven HGS (PGHGS) and Rendering Error Guided HGS (REHGS). A Gaussian is considered a hard Gaussian if it satisfies the criteria of either strategy and is observed by at least two views during the growth interval $M$ [S1].

1.  **Standard Baseline Check:**
    Standard 3DGS deems a Gaussian significant if its average view-space positional gradient over $M$ iterations satisfies [S1]:
    $$
    \frac{\sum_{j} \|g_{i,j}\|}{n} \ge \tau_{grad}
    $$
    where $n$ is the number of viewpoints $G_i$ participates in during $M$ iterations, and $\tau_{grad}$ is the gradient threshold [S1]. HGS operates because this average operation smooths large individual gradients, hindering the growth of defective Gaussians [S1].

2.  **Positional Gradient Driven HGS (PGHGS):**
    This strategy uncovers hard Gaussians by analyzing the magnitude of individual view-space gradients rather than their average [S1].
    *   For each Gaussian $G_i$, capture the $k$ largest positional gradients observed across all viewpoints during the growth interval $M$ [S1].
    *   Let these gradients be $g_{i, (1)}, g_{i, (2)}, \dots, g_{i, (k)}$ sorted by magnitude such that $\|g_{i, (1)}\| \ge \|g_{i, (2)}\| \ge \dots$ [S1].
    *   If the minimum of these $k$ gradients exceeds the threshold, i.e., $\|g_{i, (k)}\| \ge \tau_{grad}$, then $G_i$ is a hard Gaussian [S1].
    *   This exploits cross-view significant gradients to alleviate rendering inconsistency [S1].

3.  **Rendering Error Guided HGS (REHGS):**
    This strategy identifies potentially over-large Gaussians causing blurring artifacts and links them to local rendering errors [S1].
    *   **Identify Potentially Over-Large Gaussians:**
        For each Gaussian $G_i$, determine its rendered index contribution set $S_i = \{u \mid \text{idx}(u) = i, u \in P\}$, where $\text{idx}(u) = \arg \max_i w_i$ is the Gaussian with maximum contribution at pixel $u$ [S1].
        $G_i$ is potentially over-large if:
        $$
        \|S_i\| > \tau_{large} \|P\|
        $$
        where $\|P\|$ is the total number of pixels in the image and $\tau_{large}$ is the size threshold [S1].
    *   **Verify with Rendering Errors:**
        For a potentially over-large Gaussian $G_i$, project it into training viewpoint $j$ to obtain the corresponding pixel $u_{i,j}$ [S1].
        Calculate the local Structural Similarity Index Measure (SSIM) between the ground truth image $I_j$ and the rendered image $\hat{I}_j$ at $u_{i,j}$ [S1].
        $G_i$ is a hard Gaussian if:
        $$
        \text{SSIM}(I_j, \hat{I}_j)(u_{i,j}) < \tau_{SSIM}
        $$
        [S1].
    *   **Multi-View Consistency:**
        The Gaussian must satisfy the SSIM condition in at least two different views during the $M$ iterations to avoid unstable growth from single-view noise [S1].

**Parameters:**
*   $\tau_{grad}$: Gradient threshold for Gaussian growing. Maintained same as in Scaffold-GS [S1].
*   $M$: Growing interval for Gaussian growing. Maintained same as in Scaffold-GS [S1].
*   $k$: Index of the largest positional gradient used in PGHGS. Set to 3 [S1].
*   $\lambda$: Constant controlling the extent of excavation in PGHGS. Set to 1.0 [S1].
*   $\tau_{large}$: Threshold to control the extent of potentially large areas for identifying potentially over-large Gaussians. Set to $2 \times 10^{-4}$ [S1].
*   $\tau_{SSIM}$: Threshold to judge rendering quality for identifying hard Gaussians via SSIM. Set to 0.7 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def identify_hard_gaussians(gaussians, views, ground_truth_images, rendered_images, M, tau_grad, tau_large, tau_ssim, k=3):
    """
    Identifies hard Gaussians for densification based on PGHGS and REHGS criteria.
    
    Args:
        gaussians: List of Gaussian objects, each having attributes:
                   - id: unique identifier
                   - gradients: dict mapping view_id -> gradient_vector (view-space positional gradient)
                   - contributions: dict mapping view_id -> set of pixel_indices where this Gaussian has max weight
        views: List of view objects, each having:
               - id: unique identifier
               - ground_truth: ground truth image tensor
               - rendered: rendered image tensor
        M: Number of iterations in the current growth interval.
        tau_grad: Gradient magnitude threshold.
        tau_large: Fraction of image pixels threshold for over-large detection.
        tau_ssim: SSIM threshold for error detection.
        k: Number of top gradients to consider in PGHGS.
        
    Returns:
        set: Set of Gaussian IDs identified as hard Gaussians.
    """
    hard_gaussian_ids = set()
    
    # Pre-calculate image dimensions
    image_size = views[0].ground_truth.shape[1] * views[0].ground_truth.shape[2] # Assuming H, W
    
    for g in gaussians:
        is_hard = False
        
        # --- Strategy 1: Positional Gradient Driven HGS (PGHGS) ---
        # Collect all view-space positional gradient magnitudes for this Gaussian
        grad_magnitudes = []
        for view in views:
            if view.id in g.gradients:
                grad_magnitudes.append(norm(g.gradients[view.id]))
        
        # Sort magnitudes in descending order
        grad_magnitudes.sort(reverse=True)
        
        # Check if the k-th largest gradient exceeds threshold
        # If fewer than k views observed, we cannot apply the strict k-th min check 
        # as defined in the source (source implies k largest are captured).
        # We assume if n < k, we check the minimum of available gradients or skip?
        # Source says: "captures k largest... If the minimum of these k gradients exceeds..."
        # This implies we need at least k observations.
        if len(grad_magnitudes) >= k:
            kth_largest_grad = grad_magnitudes[k-1]
            if kth_largest_grad >= tau_grad:
                is_hard = True
        
        # --- Strategy 2: Rendering Error Guided HGS (REHGS) ---
        # Check for potentially over-large Gaussians
        over_large_count = 0
        error_detected_views = 0
        
        for view in views:
            # Get contribution set for this Gaussian in this view
            # Note: In practice, contributions are tracked during rasterization.
            # Here we assume g.contributions[view.id] is the set of pixel indices.
            if view.id not in g.contributions:
                continue
                
            contribution_set = g.contributions[view.id]
            num_pixels = len(contribution_set)
            
            # Check size threshold
            if num_pixels > tau_large * image_size:
                # It is potentially over-large. Now check rendering error.
                # Project Gaussian to view to find representative pixel?
                # Source says: "project it into the current training viewpoint j to obtain the corresponding pixel u_{i,j}"
                # Usually, this is the centroid or the pixel with max contribution.
                # For implementation, we can pick the pixel with max weight or the centroid.
                # Let's assume we have a function to get the primary pixel.
                u_ij = get_primary_pixel(g, view) 
                
                if u_ij is not None:
                    # Calculate local SSIM
                    # SSIM is calculated on a local window around u_ij
                    ssim_val = calculate_local_ssim(
                        ground_truth_images[view.id], 
                        rendered_images[view.id], 
                        u_ij
                    )
                    
                    if ssim_val < tau_ssim:
                        error_detected_views += 1
        
        # Must be seen by at least two views with errors to be hard via REHGS
        if error_detected_views >= 2:
            is_hard = True
            
        if is_hard:
            hard_gaussian_ids.add(g.id)
            
    return hard_gaussian_ids

def calculate_local_ssim(img_gt, img_pred, pixel_idx):
    """
    Calculates SSIM on a local window around pixel_idx.
    Returns a float between 0 and 1.
    """
    # Implementation detail: Extract local patches (e.g., 11x11)
    # Compute SSIM index
    pass

def get_primary_pixel(gaussian, view):
    """
    Returns the pixel coordinate corresponding to the projection of the Gaussian mean.
    """
    # Project mean vector mu_i using view projection matrix
    pass
```

**Edge Cases and Pitfalls:**
*   **False Positives in REHGS:** Using only the potentially over-large condition ($\|S_i\| > \tau_{large} \|P\|$) results in false positives because low-textured areas and repetitive textures may also be described by large Gaussians [S1]. The SSIM check is required to filter these [S1].
*   **Single-View Noise:** Hard Gaussians determined by only a single view can cause unstable growth. The method requires detection in at least two views during the $M$ iterations [S1].
*   **Gradient Smoothing:** Standard 3DGS averaging hides large individual gradients. PGHGS specifically targets these by looking at the $k$-th largest gradient [S1].

## Subtopics

### [PGHGS](10618-pghgs.md)
PGHGS (Positional Gradient driven Hard Gaussian Splatting) is a densification strategy that identifies "hard Gaussians" by analyzing multi-view positional gradients to alleviate cross-view rendering inconsistencies. Unlike standard methods that rely on averaged gradients which may obscure local errors, PGHGS selects Gaussians based on the magnitude of their largest individual view-space gradients, ensuring that regions with significant reconstruction errors are prioritized for growth.

### [PGHGS](10618-pghgs.md)
(Note: The source does not provide a separate child article link for REHGS, but describes it as a component of HGS. Based on the structure, REHGS is the other main variant. If no child article exists, this section might be empty or refer to the main article. However, the prompt asks for children. The source lists PGHGS as a subtopic. It does not explicitly list a separate article for REHGS in the "Children" list provided in the prompt's context, but the prompt structure implies children exist. I will list PGHGS as the only explicit child provided in the notes. If REHGS has no child article, it is not listed here.) *Correction*: The prompt lists "Children (for Subtopics): - [PGHGS]...". It does not list REHGS as a child. Therefore, only PGHGS is listed.

## Operating characteristics
*   **Time Complexity:** The method adds overhead during the training interval $M$. PGHGS requires sorting gradients for each Gaussian ($O(N \log N)$ or $O(N)$ with selection) and checking thresholds. REHGS requires iterating over potentially over-large Gaussians, projecting them, and calculating local SSIM for detected views. SSIM calculation is $O(W \times H)$ per window, but only performed for candidates. Overall, it increases training time compared to standard 3DGS due to these additional checks and SSIM computations.
*   **Memory Complexity:** Requires storing gradients per view per Gaussian and contribution sets. This increases memory usage compared to standard 3DGS which only stores average gradients or simple counters.
*   **Measured Performance:** On Tanks&Temples, HGS achieves superior NVS results. For REHGS specifically, varying $\tau_{SSIM}$ shows: $\tau_{SSIM}=0.6$ yields PSNR 26.34, SSIM 0.880, LPIPS 0.127; $\tau_{SSIM}=0.7$ yields PSNR 26.36, SSIM 0.880, LPIPS 0.126; $\tau_{SSIM}=0.8$ yields PSNR 26.39, SSIM 0.880, LPIPS 0.126 [S1].
*   **Strengths:** Resolves blurring and needle-like artifacts; recovers more complete 3D scenes; improves rendering quality in challenging areas with few Gaussians [S1].
*   **Limitations:** REHGS tends to find noticeable rendering errors but cannot detect some inconspicuous artifacts, which PGHGS probes better. Thus, PGHGS brings more obvious improvement in some cases [S1].
*   **When to use:** When standard 3DGS exhibits strong spurious artifacts, blurring, or incomplete reconstruction in challenging areas [S1].

## Background
Hard Gaussian Splatting (HGS) was proposed to address the limitations of 3D Gaussian Splatting (3DGS) in handling defective Gaussians that cause artifacts due to the smoothing effect of average gradient-based growing criteria [S1]. The method was introduced in the paper "Pushing Rendering Boundaries: Hard Gaussian Splatting" [S1]. It builds upon Scaffold-GS, maintaining the same threshold $\tau_{grad}$ and growing interval $M$ [S1]. The code is available at https://github.com/GhiXu/HGS [S1].

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
