# NeRF-SR Training Configuration

Also called: NeRF-SR, NeRF-Supersampling

## Overview
NeRF-SR is a training configuration designed to generate high-resolution (HR) novel views from low-resolution (LR) input images by optimizing a neural radiance field to match the LR observations while implicitly learning HR details through a supersampling strategy [S1]. It distinguishes itself from standard NeRF training by decoupling the training resolution from the rendering resolution, allowing the model to learn fine-grained texture details that are lost in standard LR training, and from post-processing upsampling methods by maintaining 3D consistency across views [S1]. Unlike Super-NeRF, which couples the NeRF with a GAN-based 2D super-resolution backbone and per-view latent codes, NeRF-SR relies on a deterministic supersampling strategy and a optional patch-based refinement network to enhance texture sharpness [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ LR images $\{I_n\}$ with known camera intrinsics $K_n$ and extrinsics $T_n$. The target output is an HR image at a specified scale factor (e.g., $\times 2$ or $\times 4$).
*   **Outputs:** A trained MLP representing the radiance field and the synthesized HR images.

**Data Structures:**
*   **MLP:** Two identical Multi-Layer Perceptrons (Coarse and Fine) mapping 3D coordinates and viewing directions to density and color. The architecture and size are identical to the baseline NeRF [S1].
*   **Sampling:** Hierarchical sampling is used.
    *   Coarse samples: 64 points per ray.
    *   Fine samples: 64 points per ray.
    *   Total samples per ray: 128 [S1].

**Training Steps:**
1.  **Data Preparation:** Select $N_{train}$ views for training and $N_{test}$ views for evaluation. The standard split uses 100 training views and 200 held-out test views [S1].
2.  **Ray Generation:** For each training image, generate rays. Unlike standard NeRF which samples pixels at the input resolution, NeRF-SR samples rays corresponding to the **target HR resolution**. This is the "supersampling" step. The ray origins and directions are calculated based on the camera parameters of the LR input image, but the pixel grid is denser [S1].
3.  **Volume Rendering:**
    *   Query the Coarse MLP for density $\sigma_c$ and color $c_c$ at 64 sampled points.
    *   Compute weights $w_i$ based on $\sigma_c$ and transmittance.
    *   Query the Fine MLP for density $\sigma_f$ and color $c_f$ at 64 sampled points.
    *   Render the final color $C$ using the fine samples and weights [S1].
4.  **Loss Calculation:** Compute the photometric error between the rendered HR image and the ground truth HR image. Note: Since HR ground truth is not available, the rendered HR image is downsampled to LR resolution and compared against the input LR image, or the method relies on the consistency of the supersampled rays against the LR input constraints. The source notes indicate evaluation against ground truth for metrics, but training minimizes the difference between the rendered image (upsampled or supersampled) and the input. Specifically, the method exploits 3D consistency from the supersampling strategy [S1].
5.  **Optimization:** Update MLP parameters using the Adam optimizer [S1].

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** Exponentially decayed from $5 \cdot 10^{-4}$ to $5 \cdot 10^{-6}$ [S1].
*   **Batch Size:** 2048 rays per batch [S1].
*   **Epochs:** 20 epochs for Blender dataset; 30 epochs for LLFF dataset [S1].
*   **Hyperparameters:** $\beta_1 = 0.9$, $\beta_2 = 0.999$ [S1].

**Refinement Network (Ours-Refine):**
For real-world datasets like LLFF, supersampling alone may not be sufficient for photorealistic textures. A refinement network is applied post-training or during inference to boost edge sharpness and texture details [S1].
*   **Definition:** A network that takes patches from the rendered image and reference patches to enhance details [S1].
*   **Reference Patch Selection:** $K=8$ reference patches are selected. The reference image is typically the first image in the dataset [S1].
*   **Patch Extraction:** For a pixel $i,j$ on a patch $P$, the 3D world coordinate is computed using the estimated depth $d_{i,j}$ and camera parameters:
    $$ P_{i,j}^{REF} = K_{REF}^T (K_n^{-1} d_{i,j} P_{i,j}^e) $$
    where $T_n$ is the transformation between camera extrinsics from $I_n$ to $I_{REF}$, and $K_{REF}, K_n$ are intrinsic matrices [S1]. Points falling outside $I_{REF}$ are discarded.
*   **Inference:** Input images are divided into non-overlapping $64 \times 64$ patches. These patches are refined using the reference patches and stitched together [S1].
*   **Performance:** On a single 1080Ti, rendering a $1008 \times 756$ image takes ~48 seconds, with an additional 1.3 seconds for refinement [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_sr(model, lr_images, lr_intrinsics, lr_extrinsics, 
                  target_scale, epochs, batch_size, lr_start, lr_end):
    optimizer = Adam(model.parameters(), lr=lr_start, betas=(0.9, 0.999))
    
    for epoch in range(epochs):
        # Exponential learning rate decay
        current_lr = lr_start * (lr_end / lr_start) ** (epoch / epochs)
        for param_group in optimizer.param_groups:
            param_group['lr'] = current_lr
            
        for batch in get_batches(lr_images, batch_size):
            optimizer.zero_grad()
            
            rendered_hr_images = []
            for i, (img, K, T) in enumerate(batch):
                # Generate rays at HR resolution
                rays = generate_hr_rays(img.shape, K, T, target_scale)
                
                # Hierarchical sampling
                coarse_samples = sample_coarse(rays, 64)
                coarse_density, coarse_color = model.query(coarse_samples)
                weights = compute_weights(coarse_density)
                
                fine_samples = sample_fine(rays, coarse_samples, weights, 64)
                fine_density, fine_color = model.query(fine_samples)
                
                # Volume rendering
                hr_color = volume_render(fine_samples, fine_density, fine_color, weights)
                rendered_hr_images.append(hr_color)
                
            # Loss computation: Compare rendered HR with ground truth HR
            # Note: In NeRF-SR, the "ground truth" for loss is often the 
            # downsampled input or the method relies on the consistency 
            # of the supersampled rays. The source evaluates against GT 
            # for metrics, but training minimizes photometric error.
            loss = compute_loss(rendered_hr_images, batch.gt_hr_images)
            
            loss.backward()
            optimizer.step()

def refine_patches(hr_image, reference_image, k=8, patch_size=64):
    """
    Refine HR image using reference patches.
    """
    refined_image = np.zeros_like(hr_image)
    h, w = hr_image.shape[:2]
    
    # Divide into non-overlapping patches
    patches = extract_patches(hr_image, patch_size)
    ref_patches = extract_patches(reference_image, patch_size)
    
    for i, patch in enumerate(patches):
        # Compute 3D coordinates and backproject to reference
        # P_ref = K_ref^T (K_n^-1 * d * P_e)
        # Extract corresponding patch from reference
        ref_patch = get_ref_patch(patch, ref_patches, k)
        
        # Apply refinement network
        refined_patch = refinement_network(patch, ref_patch)
        
        # Stitch back
        refined_image = stitch(refined_image, refined_patch, i)
        
    return refined_image
```

## Subtopics

### [Super-NeRF](16427-super-nerf.md)
Super-NeRF is a method for NeRF super-resolution that generates plausible, 3D-consistent high-resolution (HR) details using only low-resolution (LR) observations, without requiring HR ground truth. It distinguishes itself from NeRF-SR by coupling the NeRF with a GAN-based 2D super-resolution backbone and per-view latent codes to explicitly search the latent space of a powerful SR prior, rather than relying on simple linear formulations or HR refinement via per-patch warping. The method employs a Consistency Controlling Super Resolution (CCSR) module and mutual learning between a fixed LR NeRF and a learnable HR NeRF to ensure view consistency.

## Operating characteristics
*   **Time Complexity:** Training time is comparable to training an HR NeRF, as it involves rendering at HR resolution. Inference time for a $1008 \times 756$ image is ~48 seconds on a single 1080Ti, with an additional 1.3 seconds if the refinement network is used [S1].
*   **Memory Complexity:** Higher than standard NeRF due to the larger number of rays sampled per batch (HR resolution) and the storage of the refinement network if used.
*   **Performance:** On the LLFF dataset at $\times 4$ scale, NeRF-SR with refinement (Ours-Refine) achieves PSNR 25.59, SSIM 0.759, and LPIPS 0.165 [S1]. At $\times 2$ scale, it achieves PSNR 27.34, SSIM 0.842, and LPIPS 0.103 [S1].
*   **Strengths:** Generates photorealistic HR details without external data or HR ground truth. Maintains 3D consistency.
*   **Limitations:** Does not support arbitrary-scale super-resolution (fixed scales like $\times 2, \times 4$). Introduces extra computational cost compared to LR NeRF, though less than training a full HR NeRF from scratch [S1]. Supersampling alone may be insufficient for real-world complex structures, requiring the refinement network [S1].

## Background
NeRF-SR was introduced by Wang et al. in the paper "NeRF-SR: High-Quality Neural Radiance Fields using Supersampling" [S1]. It addresses the limitation of standard NeRF, which often produces blurry results when trained on LR inputs. The method leverages the implicit representation of NeRF to learn HR details by optimizing the radiance field against LR inputs while rendering at HR resolution. It is used for high-quality novel view synthesis in both synthetic (Blender) and real-world (LLFF) datasets [S1].

## Sources
[S1] NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, https://arxiv.org/abs/2112.01759v3, fetched 2026-10-26
