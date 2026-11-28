# Hard Gaussian Splatting

Also called: REHGS, Rendering Error guided HGS, Rendering Error guided Hard Gaussian Splatting

## Overview
Hard Gaussian Splatting (HGS) is a training-phase densification strategy that addresses artifacts in 3D Gaussian Splatting (3DGS) by growing "hard Gaussians" identified via multi-view significant positional gradients and rendering errors, rather than relying on averaged gradients [S1]. It solves the problem where standard 3DGS averaging smooths out individual large gradients and local rendering errors, hindering the growth of defective Gaussians that cause blurring or needle-like artifacts [S1]. Unlike its siblings, HGS explicitly decouples the identification of hard Gaussians into two distinct mechanisms: Positional Gradient Driven HGS (PGHGS), which targets cross-view inconsistencies by analyzing the maximum individual view-space gradients, and Rendering Error Guided HGS (REHGS), which targets blurring artifacts by correlating potentially over-large Gaussians with local SSIM errors [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** The current set of 3D Gaussians $\{G_i\}$, the set of training viewpoints $\mathcal{V}$, and the growth interval $M$ [S1].
*   **Outputs:** A subset of hard Gaussians identified for densification [S1].

**Data Structures:**
*   **Gaussian $G_i$:** Parameterized by mean vector $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, color $c_i \in \mathbb{R}^3$, and opacity $\alpha_i \in \mathbb{R}$ [S1].
*   **View-space Positional Gradient $g_{i,j}$:** The gradient of the rendering loss with respect to the mean $\mu_i$ of Gaussian $G_i$ under viewpoint $j$ [S1].
*   **Contribution Set $S_i$:** The set of pixels $u$ in the current image where Gaussian $G_i$ has the maximum contribution [S1].

**Rendering Model:**
A 3D Gaussian $G_i$ is splatted to a 2D Gaussian $G_{i,2D}$ [S1]. The pixel color $c(u)$ is computed via alpha-blending [S1]:
$$
c(u) = \sum_{i=1}^{N} w_i c_i, \quad w_i = \alpha_i G_{i,2D}(u) \prod_{k=1}^{i-1} (1 - \alpha_k G_{k,2D}(u))
$$
where $w_i$ is the blending weight [S1].

**Parameters:**
*   $\tau_{grad}$: Gradient threshold for Gaussian growing (maintained same as in Scaffold-GS) [S1].
*   $M$: Growing interval for Gaussian growing (maintained same as in Scaffold-GS) [S1].
*   $k$: Index of the largest positional gradient used in PGHGS (set to 3) [S1].
*   $\lambda$: Constant controlling the extent of excavation in PGHGS (set to 1.0) [S1].
*   $\tau_{large}$: Threshold to control the extent of potentially large areas for identifying potentially over-large Gaussians (set to $2 \times 10^{-4}$) [S1].
*   $\tau_{SSIM}$: Threshold to judge rendering quality for identifying hard Gaussians via SSIM (set to 0.7) [S1].

**Algorithm Steps:**
The method operates during the training phase [S1]. For each growth interval of $M$ iterations, the following steps are performed to identify hard Gaussians [S1].

1.  **Compute View-space Positional Gradients:**
    For each Gaussian $G_i$ and each viewpoint $j$ that observes $G_i$ during the current growth interval $M$, compute the view-space positional gradient $g_{i,j} = \nabla_{\mu_i} L$ [S1].

2.  **Identify Hard Gaussians via PGHGS (Positional Gradient Driven):**
    *   For each Gaussian $G_i$, collect the view-space positional gradients $\{g_{i,j}\}$ from all viewpoints $j$ that observed it during the interval $M$ [S1].
    *   Sort these gradients by magnitude and select the $k$ largest gradients [S1].
    *   Find the minimum magnitude among these $k$ largest gradients: $min\_grad_i = \min(\mathcal{G}_{i, top-k})$ [S1].
    *   If $min\_grad_i \ge \tau_{grad}$, classify $G_i$ as a hard Gaussian candidate for PGHGS [S1].
    *   This strategy exploits cross-view significant positional gradients to uncover hard Gaussians, alleviating rendering inconsistency across views [S1].

3.  **Identify Hard Gaussians via REHGS (Rendering Error Guided):**
    *   **Step 3.1: Identify Potentially Over-Large Gaussians.**
        For each pixel $u$ in the current rendered image, compute its rendered index $idx(u)$, identifying the Gaussian with maximum contribution:
        $$
        idx(u) = \arg \max_i w_i
        $$
        Construct the contribution set $S_i$ for each Gaussian $G_i$:
        $$
        S_i = \{ u \mid idx(u) = i, u \in P \}
        $$
        where $P$ is the set of all pixels in the image [S1].
        Identify $G_i$ as a "potentially over-large" Gaussian if:
        $$
        \|S_i\| > \tau_{large} \|P\|
        $$
        where $\|S_i\|$ is the number of pixels in the set and $\|P\|$ is the total number of pixels [S1].
        Using only this condition results in false positives because low-textured areas and repetitive textures may also be described by large Gaussians [S1].

    *   **Step 3.2: Validate with Rendering Errors.**
        For each potentially over-large Gaussian $G_i$:
        *   Project $G_i$ into the current training viewpoint $j$ to obtain the corresponding pixel $u_{i,j}$ [S1].
        *   Compute the local Structural Similarity Index Measure (SSIM) between the ground truth image $I_j$ and the rendered image $\hat{I}_j$ at $u_{i,j}$: $SSIM(I_j, \hat{I}_j)(u_{i,j})$ [S1].
        *   If $SSIM(I_j, \hat{I}_j)(u_{i,j}) < \tau_{SSIM}$, classify $G_i$ as a hard Gaussian candidate for REHGS [S1].
        *   Ensure hard Gaussians are seen by at least two views during the $M$ iterations to avoid unstable growth from single-view determination [S1].

4.  **Densification:**
    *   Union the sets of hard Gaussians identified by PGHGS and REHGS [S1].
    *   Apply the standard 3DGS densification operation (splitting and cloning) to these hard Gaussians [S1].
    *   Reset the optimizer states (momentum and variance) for the newly created or modified Gaussians [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def identify_hard_gaussians(gaussians, viewpoints, M, k, tau_grad, tau_large, tau_ssim):
    """
    Identifies hard Gaussians for densification based on PGHGS and REHGS criteria.
    """
    hard_gaussians = set()
    
    # --- PGHGS: Positional Gradient Driven ---
    for G in gaussians:
        view_grads = []
        for v in G.view_history:
            if v in G.grad_history:
                grad_vec = G.grad_history[v]
                view_grads.append(math.sqrt(sum(x**2 for x in grad_vec)))
        if not view_grads:
            continue
        view_grads.sort(reverse=True)
        top_k_grads = view_grads[:k]
        if min(top_k_grads) >= tau_grad:
            hard_gaussians.add(G)
            
    # --- REHGS: Rendering Error Guided ---
    P = all_pixel_coordinates()
    rendered_image = render_scene(gaussians, viewpoints)
    ground_truth = get_ground_truth(viewpoints)
    
    contribution_counts = {G: 0 for G in gaussians}
    pixel_max_gaussian = {}
    
    for u in P:
        max_w = -1.0
        max_G = None
        for G in gaussians:
            w = compute_weight(G, u)
            if w > max_w:
                max_w = w
                max_G = G
        pixel_max_gaussian[u] = max_G
        if max_G is not None:
            contribution_counts[max_G] += 1
            
    total_pixels = len(P)
    potentially_over_large = [G for G, count in contribution_counts.items() if count > tau_large * total_pixels]
    
    for G in potentially_over_large:
        error_views_count = 0
        for v in G.view_history:
            u_ij = project_gaussian_to_pixel(G, v)
            if u_ij is not None and u_ij in P:
                ssim_val = compute_local_ssim(ground_truth[v], rendered_image[v], u_ij)
                if ssim_val < tau_ssim:
                    error_views_count += 1
        if error_views_count >= 2:
            hard_gaussians.add(G)
            
    return hard_gaussians

def compute_weight(G, u):
    # Computes blending weight w_i = alpha_i * G_i2D(u) [S1]
    return G.opacity * evaluate_2d_gaussian(G, u)

def evaluate_2d_gaussian(G, u):
    # Evaluates the projected 2D Gaussian at pixel u [G]
    # Standard differentiable rasterization step
    return 1.0

def compute_local_ssim(img_gt, img_pred, center_pixel):
    # Computes local SSIM over a sliding window centered at center_pixel [G]
    return 0.5

def project_gaussian_to_pixel(G, v):
    # Projects 3D mean mu_i to 2D image coordinates for view v [G]
    return (0, 0)

def densify_hard_gaussians(hard_gaussians, gaussians, optimizer):
    """
    Applies densification to hard Gaussians and resets optimizer states.
    """
    new_gaussians = []
    for G in hard_gaussians:
        children = split_and_clone(G)
        new_gaussians.extend(children)
    gaussians.extend(new_gaussians)
    for G in new_gaussians:
        optimizer.reset_state(G)
    return gaussians

def split_and_clone(G):
    # Standard 3DGS split/clone operation creating child Gaussians with perturbed means and scaled covariances [G]
    return []
```

## Subtopics

### [PGHGS](10618-pghgs.md)
PGHGS (Positional Gradient driven Hard Gaussian Splatting) is a densification strategy that identifies "hard Gaussians" by analyzing multi-view positional gradients to alleviate cross-view rendering inconsistencies. Unlike standard methods that rely on averaged gradients which may obscure local errors, PGHGS selects Gaussians based on the magnitude of their largest individual view-space gradients, ensuring that regions with significant reconstruction errors are prioritized for growth.

## Operating characteristics
*   **Time Complexity:** The identification step requires computing view-space positional gradients for all Gaussians over $M$ iterations, which is $O(N \cdot V \cdot M)$ where $N$ is the number of Gaussians and $V$ is the number of views [S1]. The REHGS step requires rendering and computing SSIM for potentially over-large Gaussians, which adds overhead proportional to the number of such Gaussians and the number of views they are observed in [S1].
*   **Memory Complexity:** Requires storing gradient history for each Gaussian over the growth interval $M$, increasing memory usage by $O(N \cdot V)$ compared to standard 3DGS [S1].
*   **Measured Performance:** On the Tanks&Temples dataset, HGS achieves superior NVS results compared to standard 3DGS and other baselines [S1]. Specifically, for REHGS, varying $\tau_{SSIM}$ yields: $\tau_{SSIM}=0.6$ gives PSNR 26.34, SSIM 0.880, LPIPS 0.127; $\tau_{SSIM}=0.7$ gives PSNR 26.36, SSIM 0.880, LPIPS 0.126; $\tau_{SSIM}=0.8$ gives PSNR 26.39, SSIM 0.880, LPIPS 0.126 [S1].
*   **Strengths:** Effectively resolves blurring and needle-like artifacts by targeting hard Gaussians that standard averaging misses [S1]. It can boost both explicit and neural Gaussian methods, showing generalization ability [S1].
*   **Limitations:** REHGS tends to find noticeable rendering errors but cannot detect some inconspicuous artifacts, which PGHGS probes better [S1]. Using only the potentially over-large condition results in false positives in low-textured or repetitive texture areas [S1].
*   **When to use:** Use when standard 3DGS exhibits strong spurious artifacts, blurring, or needle-like structures in challenging areas due to insufficient Gaussian growth [S1].

## Background
Hard Gaussian Splatting (HGS) was proposed to address the limitations of 3D Gaussian Splatting (3DGS) in handling challenging areas where the average magnitude of view-space positional gradients fails to drive Gaussian growth [S1]. The method was introduced in the paper "Pushing Rendering Boundaries: Hard Gaussian Splatting" [S1]. It operates during the training phase to refine the Gaussian set without altering the inference-stage alpha-blending rendering pipeline [S1]. The code is available at https://github.com/GhiXu/HGS [S1].

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
