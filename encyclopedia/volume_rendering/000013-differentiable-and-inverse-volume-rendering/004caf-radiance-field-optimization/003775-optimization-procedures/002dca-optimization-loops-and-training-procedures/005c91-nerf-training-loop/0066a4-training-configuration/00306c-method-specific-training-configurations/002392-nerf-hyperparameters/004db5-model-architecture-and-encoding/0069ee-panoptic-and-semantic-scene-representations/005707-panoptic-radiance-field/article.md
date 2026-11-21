# Panoptic Radiance Field

Also called: Panoptic Lifting

## Overview
Panoptic Lifting learns a 3D panoptic radiance field from posed RGB images and machine-generated 2D panoptic segmentation masks, lifting noisy 2D labels into a consistent 3D representation of color, density, semantics, and instance identifiers [S1]. Unlike [Panoptic NeRF](11916-panoptic-nerf.md) or [DM-NeRF](11916-panoptic-nerf.md) which require manual ground truth or coarse meshes, this method requires only pre-trained 2D segmentation outputs, making it robust to in-the-wild scenes [S1]. It sets itself apart by using a linear assignment to resolve 2D instance inconsistencies and specific gradient-stopping techniques to prevent noisy labels from corrupting geometry [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Posed RGB images $\{I\}$ and corresponding machine-generated 2D panoptic segmentation masks. Each training ray $r$ has an associated RGB color $\hat{c}_r \in \mathbb{R}^3$, semantic class $\hat{k}_r \in K$, and 2D instance ID $\hat{h}_r \in H_I$ (defined only for thing classes $\hat{k}_r \in K_T$) [S1].
*   **Outputs:** A panoptic radiance field $\Phi(x, d)$ that, for a 3D point $x \in \mathbb{R}^3$ and direction $d \in S^2$, predicts density $\sigma \in \mathbb{R}_{\ge 0}$, semantic class distribution $\kappa$ over $K$, distribution $\pi$ over surrogate identifiers $J$, and RGB color $c \in \mathbb{R}^3$ [S1]. From novel views, it renders color images and 3D-consistent panoptic masks [S1].

**Data Structures**
The representation $\Phi$ comprises four components [S1]:
1.  **Color and Density:** Leverages a TensoRF [4] representation (lines and planes).
2.  **Semantic Field:** A small MLP predicting $\kappa$.
3.  **Instance Field:** A small MLP predicting $\pi$.
4.  **Surrogate Identifiers:** A fixed set $J$ of class-agnostic, 3D-consistent identifiers [S1].

**Parameters**
*   $K$: Set of semantic classes, partitioned into thing classes $K_T$ and stuff classes $K_S$ [S1].
*   $J$: Fixed number of surrogate identifiers held by the instance MLP [S1].
*   $H_{3D}$: Set of all possible 3D instances, $H_{3D} := K_T \times J$ [S1].
*   Loss weights: $\lambda_{con} = 1.35$, $\lambda_{ins} = 1$, $\lambda_{sem} = 1$, $\lambda_{RGB} = 1$ [S1].
*   Optimizer: Adam [S1]. Learning rates: $5 \times 10^{-4}$ for MLPs, $1 \times 10^{-2}$ for TensoRF lines and planes [S1].
*   Hardware: 1 NVIDIA A6000, 450k iterations (~10 hours) [S1].

**Preprocessing: Confidence Estimation**
To handle noisy 2D labels, test-time augmentations (horizontal flip, scale, brightness, contrast) are applied to input images. The resulting segmentations are fused by segment clustering to obtain a semantic class distribution $\hat{\kappa}_r$ and a confidence estimate $w_r$ for each pixel (ray $r$) [S1].

**Training Steps**
1.  **Ray Sampling:** Sample rays $r$ from training images. Let $R_S$ be rays randomly sampled across the scene and $R_I$ be rays sampled from a single image [S1].
2.  **Forward Pass:** For each ray, query $\Phi(x, d)$ to get $\sigma, \kappa, \pi, c$.
3.  **Rendering:** Compute rendered fields using the volumetric rendering equation [S1]:
    $$ R[f |r, \sigma] := \int_0^\infty \alpha_t (r)\sigma(r_t)f (r_t, d_{r_t})dt $$
    where $\alpha_t (r) := \exp \left(-\int_0^t \sigma(r_s)ds\right)$ is the transmittance [S1].
    *   Rendered color: $c_r = R[c|r, \sigma]$ [S1].
    *   Rendered semantic distribution: $\kappa_r = R[\kappa|r, \sigma]$ [S1].
    *   Rendered instance distribution: $\pi_r = R[\pi|r, \sigma]$ [S1].
    *   **Constraint:** The semantic and instance fields must output bounded probability distributions, not logits, to prevent unbounded signals in low-density areas [S1].
4.  **Instance Assignment:** For rays in $R_I$, map the 2D instance ID $\hat{h}_r$ to a 3D surrogate identifier $j \in J$ by solving a linear assignment problem. The cost is based on the model's current predictions and the 2D masks, ensuring consistent lifting of 2D instances to 3D [S1].
5.  **Loss Computation:**
    *   **Semantic Loss ($L_{sem}$):** Cross-entropy between rendered $\kappa_r$ and 2D label $\hat{\kappa}_r$, weighted by confidence $w_r$ [S1]:
        $$ L_{sem}(R) := -\frac{1}{|R|} \sum_{r \in R} w_r \sum_{k \in K} \hat{\kappa}_r(k) \log \kappa_r(k) $$
    *   **Instance Loss ($L_{ins}$):** Log-loss of the rendered instance distribution $\pi_r$ against the mapped surrogate identifiers from the 2D instances [S1].
    *   **Segment Consistency Loss ($L_{con}$):** Guides semantic supervision to avoid fragmenting objects due to label inconsistencies [S1].
    *   **Photometric Loss ($L_{RGB}$):** L2 loss between rendered color $c_r$ and ground truth $\hat{c}_r$ [S1].
6.  **Total Loss:**
    $$ L_{tot}(R) := \lambda_{ins} L_{ins}(R_I) + \lambda_{con} L_{con}(R_I) + \lambda_{RGB} L_{RGB}(R_S) + \lambda_{sem} L_{sem}(R_S) $$
7.  **Gradient Blocking:** Stop gradients from the semantic and instance branches back to the density field $\sigma$ to prevent noisy labels from corrupting geometry [S1].
8.  **Update:** Update parameters using Adam [S1].

**Inference**
For a novel view ray $r$:
1.  Render $\kappa_r$ and $\pi_r$.
2.  Determine semantic class $k^*_r := \arg \max_{k \in K} \kappa_r(k)$ [S1].
3.  If $k^*_r \in K_T$, determine surrogate ID $j^*_r := \arg \max_{j \in J} \pi_r(j)$ [S1].
4.  Form 3D instance ID $(k^*_r, j^*_r) \in H_{3D}$ [S1].

**Scene Editing**
*   **Deletion:** Set density $\sigma$ to zero for points where the predicted instance matches the target instance [S1].
*   **Duplication:** Manipulate rays passing through a new region to query the original instance region [S1].
*   **Manipulation:** Combine deletion and duplication [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Panoptic Lifting Training Step
# Assumes TensoRF, SemanticMLP, InstanceMLP are defined
# Assumes render_volumetric(ray, sigma, field) implements the integral equation

def train_step(rays, gt_colors, gt_semantics, gt_instances, 
               semantic_mlp, instance_mlp, tenorfr_model, 
               optimizer, lambda_con=1.35, lambda_ins=1.0, 
               lambda_sem=1.0, lambda_rgb=1.0):
    
    optimizer.zero_grad()
    
    # 1. Forward Pass
    # TensoRF provides sigma and color c
    sigma, c_pred = tenorfr_model(rays.positions, rays.directions)
    
    # Semantic and Instance MLPs provide distributions
    # Note: Outputs must be bounded probabilities, not logits
    kappa_pred = semantic_mlp(rays.positions) 
    pi_pred = instance_mlp(rays.positions)
    
    # 2. Volumetric Rendering
    # Render color, semantic distribution, and instance distribution
    # R[f|r, sigma] = integral(alpha * sigma * f)
    c_rendered = render_volumetric(rays, sigma, c_pred)
    kappa_rendered = render_volumetric(rays, sigma, kappa_pred)
    pi_rendered = render_volumetric(rays, sigma, pi_pred)
    
    # 3. Loss Computation
    
    # A. Photometric Loss (on R_S - rays sampled across scene)
    # Assuming rays are split or weighted appropriately in the batch
    # L_RGB = L2(c_rendered, gt_colors)
    loss_rgb = mse_loss(c_rendered, gt_colors)
    
    # B. Semantic Loss (on R_S)
    # Weighted Cross Entropy
    # L_sem = - sum(w_r * k_hat * log(kappa_rendered))
    # Note: gt_semantics should be the distribution k_hat_r from TTA
    # and weights w_r from TTA
    loss_sem = weighted_cross_entropy(kappa_rendered, gt_semantics, rays.confidence)
    
    # C. Instance Loss (on R_I - rays from single image)
    # Map 2D instance IDs to 3D surrogate IDs via linear assignment
    # This step resolves inconsistencies
    mapped_surrogates = solve_linear_assignment(
        rays.instances, 
        pi_pred, 
        instance_mlp.num_surrogates
    )
    loss_ins = cross_entropy(pi_rendered, mapped_surrogates)
    
    # D. Segment Consistency Loss
    # Guides semantic field to avoid fragmenting objects
    loss_con = compute_segment_consistency_loss(kappa_rendered, rays.instances)
    
    # 4. Total Loss
    # Gradients from semantic and instance branches are STOPPED before reaching sigma
    # This is implemented by detaching sigma from the computation graph 
    # for the semantic/instance branches or using stop_gradient ops.
    sigma_detached = sigma.detach() # Or equivalent stop_gradient
    
    # Re-render or compute losses using detached sigma if necessary, 
    # but typically the rendering equation integrates sigma. 
    # The key is that d(L_sem)/d(sigma) = 0.
    
    total_loss = (lambda_ins * loss_ins + 
                  lambda_con * loss_con + 
                  lambda_rgb * loss_rgb + 
                  lambda_sem * loss_sem)
    
    # 5. Backpropagation
    total_loss.backward()
    optimizer.step()
    
    return total_loss.item()
```

## Operating characteristics
*   **Time Complexity:** Training is lengthy, requiring ~10 hours on a single NVIDIA A6000 for 450k iterations [S1]. It runs offline due to pre-processing (pose estimation, 2D segmentation) and optimization [S1].
*   **Memory Complexity:** Uses TensoRF for density/color (compact) plus two small MLPs for semantics and instances [S1].
*   **Performance:** Improves scene-level Panoptic Quality (PQ) by 8.4% on Hypersim, 13.8% on Replica, and 10.6% on ScanNet over state-of-the-art methods [S1]. Maintains comparable PSNR for novel view synthesis [S1].
*   **Strengths:** Robust to noisy, machine-generated 2D labels; requires no manual 3D ground truth; enables scene editing [S1].
*   **Limitations:** Limited to classes supported by the pre-trained 2D segmenter (e.g., Mask2Former); offline processing; performance depends on the quality of the 2D segmentation model [S1].

## Background
Panoptic Lifting was introduced in "Panoptic Lifting for 3D Scene Understanding with Neural Fields" (CVPR 2023) [S1]. It addresses the limitation of previous panoptic NeRF methods (like Panoptic NeRF and DM-NeRF) which required manual ground truth or coarse meshes [S1]. It builds on [TensoRF](13622-binary-radiance-fields.md) for efficient density/color representation and uses [Mask2Former](11916-panoptic-nerf.md) as the pre-trained 2D segmenter [S1].

## Sources
[S1] Panoptic Lifting for 3D Scene Understanding with Neural Fields, https://doi.org/10.1109/cvpr52729.2023.00873, fetched 2026-10-29
