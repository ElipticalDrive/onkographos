# DoF-NeRF

Also called: Depth-of-Field Meets Neural Radiance Fields

## Overview
DoF-NeRF is a neural rendering framework that extends the standard Neural Radiance Field (NeRF) to simulate the depth-of-field (DoF) effects of a physical lens, allowing it to operate on views with different focus configurations and manipulate DoF effects by adjusting virtual aperture and focus parameters [S1]. Unlike vanilla NeRF, which assumes a pinhole camera model and all-in-focus inputs, DoF-NeRF explicitly models the aperture and focus distance, enabling it to handle shallow DoF inputs and produce clear, all-in-focus novel view synthesis with reduced blurriness and enhanced texture details [S1]. It achieves this by combining optical models with classical volume rendering to approximate the DoF effect under arbitrary aperture size and focus distance configurations [S1].

## How it works
DoF-NeRF modifies the NeRF rendering pipeline to account for the Circle of Confusion (CoC) caused by finite aperture size and defocus. The method introduces two learnable parameters per training view: the aperture parameter $K$ and the focus distance $F$, which are jointly optimized with the NeRF network weights [S1]. The rendering process simulates DoF through a "concentrate-and-scatter" mechanism: radiance from spatial points scatters to a particular area (concentration), and the center ray accepts scattered radiance from neighboring rays (scatter) [S1].

### Inputs and Outputs
*   **Inputs:** A set of $N$ training images $I = \{I_1, \dots, I_N\}$ with known camera poses (obtained via COLMAP) [S1].
*   **Outputs:** A trained NeRF model $G_\Theta$ and optimized learnable parameters: aperture parameters $\hat{K} = \{\hat{K}_1, \dots, \hat{K}_N\}$ and focus distances $\hat{F} = \{\hat{F}_1, \dots, \hat{F}_N\}$ [S1].

### Data Structures and Parameters
*   **NeRF Model $G_\Theta$:** The standard MLP mapping $(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [S1].
*   **Aperture Parameter $\hat{K}$:** A learnable vector of size $N \times 1$, where $\hat{K}_n$ corresponds to the $n$-th image. Defined as $K = f \times D$, where $f$ is focal length and $D$ is aperture diameter [S1].
*   **Focus Distance $\hat{F}$:** A learnable vector of size $N \times 1$, where $\hat{F}_n$ corresponds to the $n$-th image [S1].
*   **Patch Parameters:**
    *   $N_{anchor}$: Spacing between anchors on the imaging plane.
    *   $N_{patch}$: Size of the square patch ($N_{patch} \times N_{patch}$) centered at each anchor [S1].

### Formulas
1.  **Circle of Confusion (CoC) Diameter:**
    The simplified CoC diameter $\delta(h_t)$ for a point at object distance $h_t$ is:
    $$ \delta(h_t) = K \times \left| \frac{1}{F} - \frac{1}{h_t} \right| $$
    where $K$ is the aperture parameter and $F$ is the focus distance [S1].

2.  **Concentration (Scattered Radiance at Center):**
    The scattered radiance $\hat{C}^s(p)$ at a concentration point $p$ is computed by summing contributions from $N_s$ samples along the ray $r_p$:
    $$ \hat{C}^s(p) = \frac{\sum_{i=1}^{N_s} K_{volume}(r_p, h_i) c(r_p(h_i), d)}{\pi \delta_c^2(p)/4} $$
    where $K_{volume}$ is the volume rendering weight, $c$ is the color, and $\delta_c(p)$ is the CoC radius at pixel $p$ [S1].

3.  **Scatter (Final Observed Color):**
    The final observed color $\hat{I}(p)$ for a pixel $p$ is the sum of scattered radiance from neighboring pixels $p_j$ within the CoC radius:
    $$ \hat{I}(p) = \sum_{j=1}^{P} \mathbb{1}_{\delta_c(p) > 2 \|p - p_j\|^2} \hat{C}^s(p_j) $$
    where $P$ is the number of neighboring pixels considered, and $\mathbb{1}$ is an indicator function checking if the distance between pixels is within the CoC diameter [S1].
    *Note: The source also provides an alternative formulation for scattered radiance $\hat{I}_{scatter}(p)$ summing over $M$ neighbors $p_j$ and $N_s$ samples:*
    $$ \hat{I}_{scatter}(p) = \sum_{j=1}^{M} \sum_{i=1}^{N_s} \mathbb{1}_{\delta(h'_i) > 2 \|p - p_j\|^2} T'_{ij} \hat{C}(p_j, h'_i) $$
    [S1]

4.  **Loss Function:**
    The optimization loss $L$ is the photometric error (Mean Squared Error) between synthesized views $\hat{I}$ and observations $I$, summed over all $N$ input images:
    $$ L = \sum_{n=1}^{N} \| I_n - \hat{I}_n \|_2^2 $$
    [S1]

### Pseudocode
The following pseudocode implements the two-stage optimization process described in [S1]. It assumes standard volume rendering functions and PyTorch-like tensor operations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def DoF_NeRF_Training(N_images, N_iters, N_pretrain, N_anchor, N_patch):
    """
    N_images: Number of training images
    N_iters: Total number of optimization iterations
    N_pretrain: Number of pretraining iterations (Stage 1)
    N_anchor: Spacing for anchor points on the image plane
    N_patch: Size of the patch (N_patch x N_patch)
    """
    
    # 1. Initialize Learnable Parameters
    # K and F are learnable parameters for each of the N images
    K = nn.Parameter(torch.randn(N_images, 1), requires_grad=True)
    F = nn.Parameter(torch.randn(N_images, 1), requires_grad=True)
    
    # Initialize NeRF Model (standard MLP)
    NeRF_Model = NeRF_Model_Class() # Assumes standard NeRF architecture
    
    # 2. Two-Stage Optimization
    for i in range(N_iters):
        if i < N_pretrain:
            # Stage 1: Pinhole Camera Training
            # Train NeRF with aperture parameter set to 0 (pinhole model)
            # Use random rays for efficiency
            rays, target_images = random_rays_selection(N_images)
            
            # Standard Volume Rendering (no DoF simulation)
            rendered_images = Volume_Rendering(NeRF_Model, rays)
            
            # Compute Loss
            loss = torch.mean((rendered_images - target_images) ** 2)
            
            # Update NeRF weights only
            loss.backward()
            optimizer_NeRF.step()
            optimizer_NeRF.zero_grad()
            
        else:
            # Stage 2: Joint Optimization with DoF
            # Use patch-based ray selection
            rays, target_images = patch_rays_selection(N_images, N_anchor, N_patch)
            
            # Step 1: Concentration
            # Compute scattered radiance at concentration points
            # Returns scattered radiance C_hat and CoC radius delta_c
            C_hat, delta_c = Concentration(NeRF_Model, rays, K, F)
            
            # Step 2: Scatter
            # Compute final observed color using concentrate-and-scatter
            rendered_images = Scatter(C_hat, delta_c, rays)
            
            # Compute Loss
            loss = torch.mean((rendered_images - target_images) ** 2)
            
            # Update NeRF weights, K, and F jointly
            loss.backward()
            optimizer_all.step() # Updates NeRF_Model, K, and F
            optimizer_all.zero_grad()

    return NeRF_Model, K, F

def Concentration(NeRF_Model, rays, K, F):
    """
    Simulates the concentration step: radiance scatters to a particular area.
    Computes the scattered radiance at the center of the CoC.
    """
    # Sample points along rays
    # For each ray, compute volume rendering weights and colors
    # Calculate CoC radius delta_c for each pixel based on K and F
    # Aggregate contributions into C_hat (scattered radiance at center)
    # Formula: C_hat(p) = sum(K_volume * color) / (pi * delta_c^2 / 4)
    
    # Implementation detail:
    # 1. Query NeRF for sigma and color at sampled points along rays
    # 2. Compute volume rendering weights T
    # 3. Calculate delta_c using delta(h_t) = K * |1/F - 1/h_t|
    # 4. Sum weighted colors and divide by CoC area
    
    return C_hat, delta_c

def Scatter(C_hat, delta_c, rays):
    """
    Simulates the scatter step: center ray accepts scattered radiance from neighbors.
    """
    # For each pixel p, sum C_hat from neighboring pixels p_j
    # where distance ||p - p_j||^2 < delta_c(p) / 2
    # Formula: I(p) = sum_{j} 1_{delta_c(p) > 2||p-p_j||^2} * C_hat(p_j)
    
    rendered_image = torch.zeros_like(target_images)
    for p in pixels:
        sum_radiance = 0
        for j in neighbors(p, delta_c[p]):
            if delta_c[p] > 2 * distance(p, j)**2:
                sum_radiance += C_hat[j]
        rendered_image[p] = sum_radiance
        
    return rendered_image

def patch_rays_selection(N_images, N_anchor, N_patch):
    """
    Selects rays based on a randomly chosen patch.
    Anchors are set every N_anchor pixels.
    One patch of size N_patch x N_patch is randomly chosen.
    Rays passing through pixels in that patch are returned.
    """
    # 1. Define grid of anchors
    # 2. Randomly select one anchor index
    # 3. Define patch boundaries around the anchor
    # 4. Collect rays corresponding to pixels in the patch
    # 5. Return rays and corresponding target image patches
    pass

def random_rays_selection(N_images):
    """
    Selects random rays from the N images for Stage 1 training.
    """
    # Randomly sample rays from the N images
    pass
```

### Edge Cases and Pitfalls
*   **Divergence:** Using randomly chosen rays for DoF simulation in the initial stages can lead to divergence or sub-optimal results due to the gathering of rays [S1]. This is mitigated by the two-stage optimization process [S1].
*   **Computational Cost:** Computing rays for the whole imaging plane in each iteration is computationally unacceptable [S1]. The patch-based ray selection method is used as a compromise to reduce cost [S1].
*   **Blurriness in Vanilla NeRF:** Vanilla NeRF is prone to generating blurry renderings and missing texture details when trained on shallow DoF inputs because it ignores the interference between rays and does not model the aperture structure [S1].

## Operating characteristics
*   **Time Complexity:** The method uses a two-stage optimization. Stage 1 is efficient as it uses random rays and standard volume rendering. Stage 2 uses patch-based ray selection, which is more efficient than processing the entire image plane but more complex than Stage 1 due to the concentrate-and-scatter rendering [S1].
*   **Memory Complexity:** Requires storing the NeRF model, and learnable parameters $K$ and $F$ for each training image. The patch-based selection reduces memory pressure compared to full-image processing per iteration [S1].
*   **Performance:**
    *   On the real-world dataset (amiya scene), DoF-NeRF achieves PSNR 28.311 vs NeRF 26.924, SSIM 0.9289 vs 0.9092, and LPIPS 0.1370 vs 0.1633 [S1].
    *   On the synthetic dataset (room scene), DoF-NeRF achieves PSNR 29.443 vs NeRF 26.668, SSIM 0.9135 vs 0.8743, and LPIPS 0.1502 vs 0.1961 [S1].
*   **Strengths:**
    *   Handles shallow DoF inputs effectively.
    *   Produces clear, all-in-focus novel views with fine-grained details.
    *   Allows direct manipulation of DoF effects by adjusting virtual aperture and focus parameters [S1].
    *   Plug-and-play and can be inserted into NeRF-based frameworks [S1].
*   **Limitations:**
    *   Requires joint optimization of aperture and focus distances, which adds complexity to the training process [S1].
    *   Performance depends on the quality of the initial camera parameters (obtained via COLMAP) and the accuracy of the depth estimation for synthetic data generation [S1].

## Background
DoF-NeRF was introduced by Zijin Wu, Juewen Peng, and Weicai Zhong from Huazhong University of Science and Technology and Huawei [S1]. It was presented at the ACM Multimedia Conference (MM '22) in Lisboa, Portugal, on October 10–14, 2022 [S1]. The work addresses the limitation of existing NeRF models, which assume a linear pinhole camera model and all-in-focus inputs, making them unsuitable for real-world images with finite depth-of-field [S1]. It builds upon the principles of geometric optics to simulate the aperture of a lens [S1].

## Sources
[S1] DoF-NeRF: Depth-of-Field Meets Neural Radiance Fields, https://doi.org/10.1145/3503161.3548088, fetched 2026-10-29
