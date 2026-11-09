# DoF-NeRF

Also called: Depth-of-Field Meets Neural Radiance Fields

## Overview
DoF-NeRF is a neural rendering framework that extends Neural Radiance Fields (NeRF) to simulate the depth-of-field (DoF) effect of a physical lens aperture, allowing it to operate on views with different focus configurations and manipulate DoF effects by adjusting virtual aperture and focus parameters [S1]. Unlike vanilla NeRF, which assumes a pinhole camera model and all-in-focus inputs, DoF-NeRF explicitly models the aperture and focus distance, enabling the synthesis of clear novel views from shallow DoF inputs and the generation of all-in-focus scenes from blurred inputs [S1]. It achieves this by combining optical models with classical volume rendering, specifically incorporating the Circle of Confusion (CoC) to approximate DoF under arbitrary aperture size and focus distance [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $\mathcal{I} = \{I_1, I_2, \dots, I_N\}$ with known camera parameters (generated via COLMAP or similar SfM pipelines) [S1].
*   **Outputs:** A trained NeRF model $G_\Theta$ (regressing color and density), and optimized learnable parameters for each training view: aperture parameters $\hat{K} = \{\hat{K}_1, \dots, \hat{K}_N\}$ and focus distances $\hat{F} = \{\hat{F}_1, \dots, \hat{F}_N\}$ [S1].

**Data Structures:**
*   **NeRF Model $G_\Theta$:** Standard MLP-based NeRF taking 3D coordinates and viewing directions, outputting volume density $\sigma$ and view-dependent color $\mathbf{c}$ [S1].
*   **Learnable Parameters:**
    *   $\hat{K} \in \mathbb{R}^{N \times 1}$: Aperture parameter for each of the $N$ training images. Defined as $K = f \times D$, where $f$ is focal length and $D$ is aperture diameter [S1].
    *   $\hat{F} \in \mathbb{R}^{N \times 1}$: Focus distance for each of the $N$ training images [S1].
*   **Ray Data:** Rays $\mathbf{r}$ sampled from the imaging plane, associated with target pixel coordinates $\mathbf{p}$ and observed colors $I$ [S1].

**Formulas:**
1.  **Circle of Confusion (CoC) Diameter:**
    The simplified CoC diameter $\delta(h_t)$ for a spatial point at object distance $h_t$ is given by [S1]:
    $$ \delta(h_t) = K \times \left| \frac{1}{F} - \frac{1}{h_t} \right| $$
    where $K$ is the aperture parameter and $F$ is the focus distance [S1].

2.  **Concentration (Scattered Radiance at Concentration Point):**
    For a center ray $\mathbf{r}_p$ passing through pixel $\mathbf{p}$, the scattered radiance $\hat{C}^s(p)$ is computed by aggregating contributions from neighboring rays within the CoC radius $\delta_c(p)$ [S1]:
    $$ \hat{C}^s(p) = \frac{\sum_{i=1}^{N_s} K_{volume}(\mathbf{r}_p, h_i) c(\mathbf{r}_p(h_i), d)}{\pi \delta_c^2(p)/4} $$
    where $N_s$ is the number of samples along the ray, $K_{volume}$ represents the volume rendering weight (transmittance and density), and $c$ is the color [S1]. Note: The source uses $K_{volume}$ to denote the accumulated volume rendering term for the center ray's samples [S1].

3.  **Scatter (Final Observed Color):**
    The final observed color $\hat{I}(p)$ for pixel $\mathbf{p}$ is computed by summing the concentrated radiance from neighboring pixels $\mathbf{p}_j$ that fall within the CoC diameter $\delta_c(p)$ of $\mathbf{p}$ [S1]:
    $$ \hat{I}(p) = \sum_{j=1}^{P} \mathbb{1}_{\delta_c(p) > 2 \|\mathbf{p} - \mathbf{p}_j\|^2} \hat{C}^s(p_j) $$
    where $P$ is the number of neighboring pixels considered, and $\mathbb{1}$ is an indicator function [S1].
    *Note: The source also provides an alternative formulation for scattered radiance $\hat{I}_{scatter}(p)$ involving neighboring pixels $p_j$ and samples $i$ [S1]:*
    $$ \hat{I}_{scatter}(p) = \sum_{j=1}^{M} \sum_{i=1}^{N_s} \mathbb{1}_{\delta(h'_i) > 2 \|\mathbf{p} - \mathbf{p}_j\|^2} T'_{ij} \hat{C}(p_j, h'_i) $$
    where $T'_{ij}$ is the transmittance and $\hat{C}$ is the color [S1]. The primary rendering loop uses the Concentrate-then-Scatter approach defined in equations 13-14 of the source [S1].

4.  **Loss Function:**
    The optimization loss $L$ is the photometric error between synthesized views $\hat{I}$ and observations $I$, summed over all $N$ input images [S1]:
    $$ L = \sum_{n=1}^{N} \| I_n - \hat{I}_n \|_2^2 $$

**Parameters:**
*   **Aperture Parameter $K$:** Learnable per image. Initialized or optimized jointly [S1].
*   **Focus Distance $F$:** Learnable per image. Initialized or optimized jointly [S1].
*   **Patch Size:** $N_{patch} \times N_{patch}$ pixels [S1].
*   **Anchor Spacing:** $N_{anchor}$ pixels on the imaging plane [S1].
*   **Image Resolution:** 497×331 for real-world datasets; 504×378 for synthetic datasets [S1].

**Pseudocode:**
The following pseudocode implements the two-stage joint optimization process described in Algorithm 1 of [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

# Initialize learnable parameters
# K_hat: Aperture parameters for N images, shape (N, 1)
# F_hat: Focus distances for N images, shape (N, 1)
K_hat = nn.Parameter(torch.zeros(N, 1), requires_grad=True)
F_hat = nn.Parameter(torch.zeros(N, 1), requires_grad=True)

# Initialize NeRF model
# G_Theta: NeRF MLP taking (x, y, z, dx, dy, dz) -> (sigma, color)
G_Theta = NeRF_Model(requires_grad=True)

optimizer = torch.optim.Adam(list(G_Theta.parameters()) + [K_hat, F_hat], lr=learning_rate)

for i in range(N_iters):
    # Select a training image index n
    n = random_int(0, N-1)
    I_n = training_images[n]
    
    if i < N_pretrain:
        # Stage 1: Pinhole camera training (Aperture = 0)
        # Set aperture to 0 for this stage to degenerate to pinhole model
        K_current = torch.zeros_like(K_hat)
        F_current = F_hat # Focus distance not used in pinhole, but kept for consistency
        
        # Sample random rays for the image
        rays_n, target_colors_n = random_rays(I_n)
        
        # Standard Volume Rendering (Pinhole)
        # G_Theta outputs sigma and color for each point on the ray
        # Volume rendering integrates these along the ray
        rendered_colors_n = volume_rendering(G_Theta, rays_n)
        
        # Compute Loss
        loss = torch.mean((rendered_colors_n - target_colors_n) ** 2)
        
        # Backpropagate only to NeRF parameters
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    else:
        # Stage 2: Concentrate-and-Scatter with Patch-based Ray Selection
        # Use current learned K_hat and F_hat
        K_current = K_hat[n]
        F_current = F_hat[n]
        
        # Patch-based ray selection
        # 1. Define anchors every N_anchor pixels on the imaging plane
        # 2. Randomly choose one anchor
        # 3. Select all rays passing through pixels in the N_patch x N_patch patch centered at the anchor
        rays_n, target_colors_n = patch_rays(I_n, N_anchor, N_patch)
        
        # Step 1: Concentration
        # Render the center rays of the patch using standard volume rendering
        # to get the "concentrated" radiance at the center of the CoC
        # This approximates the integral of radiance over the CoC area
        concentrated_radiance_n = concentration_step(G_Theta, rays_n, K_current, F_current)
        
        # Step 2: Scatter
        # For each pixel in the patch, compute the final color by scattering
        # the concentrated radiance from neighboring pixels within the CoC radius
        # This involves calculating the CoC radius delta_c for each pixel
        # and summing contributions from neighbors based on the indicator function
        rendered_colors_n = scatter_step(concentrated_radiance_n, rays_n, K_current, F_current)
        
        # Compute Loss
        loss = torch.mean((rendered_colors_n - target_colors_n) ** 2)
        
        # Backpropagate to NeRF, K, and F
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
```

**Edge Cases and Pitfalls:**
*   **Divergence:** Jointly optimizing aperture, focus, and NeRF geometry from scratch can lead to divergence or sub-optimal results due to the complexity of the concentrate-and-scatter rendering [S1]. The two-stage optimization (pinhole first, then DoF) mitigates this [S1].
*   **Computational Cost:** Rendering every ray on the whole imaging plane in each iteration is computationally unacceptable [S1]. Patch-based ray selection is used as a compromise [S1].
*   **Blurriness:** Vanilla NeRF on shallow DoF inputs produces blurry renderings and misses texture details because it ignores ray interference and aperture structure [S1]. DoF-NeRF addresses this by explicitly modeling these factors [S1].

## Operating characteristics
*   **Time Complexity:** The concentrate-and-scatter rendering is more expensive than standard volume rendering due to the neighbor aggregation step. However, patch-based ray selection reduces the number of rays processed per iteration compared to full-image rendering [S1].
*   **Memory Complexity:** Requires storing the NeRF model, plus $2N$ learnable parameters for $N$ training images (aperture and focus distances) [S1].
*   **Performance:**
    *   **Real-world dataset (amiya scene):** DoF-NeRF achieves PSNR 28.311 vs NeRF 26.924, SSIM 0.9289 vs 0.9092, and LPIPS 0.1370 vs 0.1633 [S1].
    *   **Synthetic dataset (room scene):** DoF-NeRF achieves PSNR 29.443 vs NeRF 26.668, SSIM 0.9135 vs 0.8743, and LPIPS 0.1502 vs 0.1961 [S1].
*   **Strengths:**
    *   Handles shallow DoF inputs effectively [S1].
    *   Enables all-in-focus novel view synthesis from blurred inputs [S1].
    *   Allows manipulation of DoF effects by adjusting virtual aperture and focus parameters [S1].
    *   Plug-and-play with NeRF-based frameworks [S1].
*   **Limitations:**
    *   Requires optimization of additional parameters (aperture, focus) per image [S1].
    *   More complex training procedure (two-stage) compared to vanilla NeRF [S1].
    *   Assumes ideal optical system (neglects ray distortions caused by the lens) [S1].

## Background
DoF-NeRF was introduced by Zijin Wu, Juewen Peng, and Weicai Zhong in 2022 [S1]. It was presented at the ACM Multimedia Conference (MM '22) in Lisboa, Portugal [S1]. The work addresses the limitation of existing NeRF variants (like RawNeRF and NeRFocus) which assume all-in-focus inputs, by explicitly modeling aperture and focus distance to handle shallow DoF inputs [S1]. It builds on the principles of geometric optics and classical volume rendering [S1].

## Sources
[S1] DoF-NeRF: Depth-of-Field Meets Neural Radiance Fields, https://doi.org/10.1145/3503161.3548088, fetched 2026-10-29
