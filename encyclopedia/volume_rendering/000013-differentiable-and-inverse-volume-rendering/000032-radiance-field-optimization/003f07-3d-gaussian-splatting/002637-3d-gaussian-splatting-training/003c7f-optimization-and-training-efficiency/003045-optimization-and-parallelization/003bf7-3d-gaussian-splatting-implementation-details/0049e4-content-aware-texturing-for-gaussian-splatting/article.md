# Content-Aware Texturing for Gaussian Splatting

Also called: Content-Aware Texturing, Ours

## Overview
This method augments 2D Gaussian primitives with spatially varying texture maps that add offsets to spherical harmonic colors, allowing appearance to adapt to captured content without pretrained models [S1]. It distinguishes itself from sibling methods by using dynamic, content-aware texel allocation and splitting-based densification to manage geometric and appearance errors, avoiding the convergence issues associated with cloning textured primitives [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Set of 2D Gaussian primitives (position $\mu$, scale $\sigma$, rotation $R$, opacity $\alpha$, SH coefficients).
*   Input views $\Pi$ with ground truth images.
*   Camera parameters for each view.

**Outputs:**
*   Optimized primitive parameters (including per-primitive texel size $k_i$).
*   Texture maps $T_i$ for each primitive (stored in a jagged tensor structure).
*   Rendered images.

### Data Structures
**Primitive Parameters:**
Each primitive stores 59 parameters, each as a 4-byte float [S1]:
*   Position: 3 floats ($\mathbb{R}^3$).
*   Scale: 2 floats (for 2D Gaussian, representing extent in local axes).
*   Rotation: 4 floats (quaternion).
*   Opacity: 1 float.
*   Color: 48 floats (Spherical Harmonics coefficients).
*   Texel Size: 1 float ($k_i$).

**Texture Storage:**
*   Each texel stores 3 floats (RGB offsets).
*   Stored in a custom "jagged" tensor data structure to handle varying resolutions per primitive [S1].
*   Texel values are constrained to $[-1, 1]$ via sigmoid activation on unactivated features $c'_{Ti}$:
    $$ c^T_i = 2\sigma(c'_{Ti}) - 1 $$
    where $\sigma$ is the sigmoid function [S1].

### Coordinate Systems
To facilitate operations, the method uses four coordinate systems [S1]:
1.  **World Space ($p_w$):** Intersection point in world coordinates.
2.  **World Space Centered ($p_{w0}$):** $p_{w0} = p_w - \mu$.
3.  **Local Axis-Aligned Space ($p_l$):** $p_l = R^{-1} p_{w0}$. Since the last component is 0, $p_l \in \mathbb{R}^2$.
4.  **Local Normalized/Canonical Space ($p_c$):** $p_c = S^{-1} p_l$, where $S = \text{diag}(\sigma)$.

### Rendering Formula
The color of a ray $r$ is computed by alpha blending [S1]:
$$ C(r) = \sum_i w_i(p_i) c_i(d) = \sum_i T_i o_i G_i(p_i) c_i(d) $$
$$ T_i = \prod_{j<i} (1 - o_j G_j(p_j)) $$
where:
*   $w_i(p_i)$ is the contribution of Gaussian $i$.
*   $G(x) = e^{-\frac{1}{2} (x^T x)}$ is the Gaussian falloff.
*   $T_i$ is the transmittance before primitive $i$.
*   $o_i$ is the opacity.
*   $c_i(d)$ is the color of primitive $i$ for direction $d$.

The color $c_i$ combines spherical harmonics and texture [S1]:
$$ c_i = \text{SH}(d) + \text{bilerp}(T_i, u) $$
where $u$ are the UV coordinates in texture space [S1]:
$$ u = \frac{p_l}{k_i} + T_{\text{offset}} $$
*   $k_i$ is the texel size in world space units.
*   $T_{\text{offset}}$ centers the texture map.
*   $p_l$ is the intersection point in local axis-aligned space.

### Texture Resolution and Scaling
The texel size $k$ is determined by a minimum pixel size $k_{\min}$ and a ratio $t2_{pr}$ [S1]:
$$ k = k_{\min} \cdot t2_{pr} $$
$t2_{pr}$ takes values that are powers of two, with a lower bound of 1. The method allows upscaling only for primitives with $t2_{pr} \ge 2$ [S1].

### Training Loss
The total loss $L$ is the sum of RGB, texture sparsity, and opacity regularization terms [S1]:
$$ L = L_{\text{RGB}} + L_{\text{texture}} + L_{\text{opacity}} $$

1.  **RGB Loss:** Weighted sum of L1 and SSIM [S1]:
    $$ L_{\text{RGB}} = (1 - \lambda_{\text{SSIM}}) L_1 + \lambda_{\text{SSIM}} L_{\text{SSIM}} $$
    Typical value: $\lambda_{\text{SSIM}} = 0.2$ [S1].

2.  **Texture Sparsity Loss:** Pushes texel values toward zero to allow downsampling [S1]:
    $$ L_{\text{texture}} = \lambda_{\text{texture}} \sum_i |c^T_i| $$

3.  **Opacity Regularization:** Incentivizes low-contribution primitives to disappear [S1]:
    $$ L_{\text{opacity}} = \lambda_{\text{opacity}} \frac{1}{N} \sum_i o_i $$

### Primitive Management and Densification
The method uses **splitting** instead of cloning to avoid superimposition issues that hinder convergence [S1].

**Error Calculation:**
Per-primitive error $E_i$ is a weighted sum of per-view errors, proportional to the primitive's contribution [S1]:
$$ E_i = \frac{\sum_{\pi \in \Pi} E_i^\pi w_i^\pi}{\sum_{\pi \in \Pi} w_i^\pi} $$
where $w_i^\pi = \sum_{r \in P_i} w_i^\pi(r)$ and $P_i$ is the set of pixels covered by primitive $i$ in view $\pi$ [S1].

**Upscaling (Appearance Error):**
1.  Select the top 10% of primitives with highest error $E_i$ [S1].
2.  If $t2_{pr} \ge 2$, halve $t2_{pr}$ (upscale texture by factor of 4) [S1].

**Downscaling (Low Frequency):**
1.  Compute error $E_d$ between original texture $T_{\text{orig}}$ and low-pass filtered version $T_{\text{lowpass}}$, weighted by Gaussian falloff [S1]:
    $$ E_d = \frac{\sum_p G(p)(T_{\text{orig}}(p) - T_{\text{lowpass}}(p))}{\sum_p G(p)} $$
2.  If $E_d < \tau_{ds}$, double $t2_{pr}$ (downscale) [S1].

**Splitting (Geometric Error):**
1.  Select the top 10% of primitives with highest error $E_i$ [S1].
2.  Check if texture resolution exceeds threshold $\tau_{tr}$ [S1].
3.  Replace primitive with two new primitives, displaced by $\pm 1$ standard deviation along the axis [S1].
4.  New primitives have:
    *   Half the scale in the corresponding axis.
    *   Half the texture resolution.
    *   $G(1)$ times the opacity.
5.  Initialize texture map by sampling the original point at respective locations [S1].

### Edge Cases and Pitfalls
*   **Texture Stretching:** Naive mapping fixes relative texture coordinates, causing stretching when primitives scale/rotate. The world-space fixed texel size $k_i$ prevents this [S1].
*   **Cloning Incompatibility:** Cloning creates superimposed textured primitives, making convergence difficult and requiring excessive parameters. Splitting is used instead [S1].
*   **Primitive Growth:** If a primitive grows outside its allocated texture grid (due to lack of allocation or hitting the 256 limit), zero-padding is used. This reduces to the DC color since texels are offsets [S1].
*   **Memory Limit:** Hard limit of 256 texture resolution to avoid excessive memory usage [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_ray(ray, primitives, camera_params):
    """
    Renders a single ray using alpha blending with content-aware textures.
    """
    # Transform ray to camera space (implementation details depend on camera model)
    # For each primitive, compute intersection and contribution
    
    transmittance = 1.0
    color = [0.0, 0.0, 0.0]
    
    # Sort primitives by depth along ray for correct alpha blending
    sorted_primitives = sort_primitives_by_depth(primitives, ray)
    
    for prim in sorted_primitives:
        # 1. Compute intersection point p_w in world space
        p_w = intersect_ray_prim(ray, prim)
        if p_w is None:
            continue
            
        # 2. Transform to local axis-aligned space p_l
        p_w0 = p_w - prim.mu
        p_l = inverse(prim.R) @ p_w0[:2] # Take first 2 components
        
        # 3. Compute Gaussian weight G(p_i)
        # G(x) = exp(-0.5 * x^T x)
        # Note: p_l is in local axis-aligned space, but G is evaluated in canonical space for standard 2DGS
        # However, Eq 2 uses G(p_i) where p_i is the intersection. 
        # The paper states G(x) = exp(-0.5 (x^T x)). 
        # In 2DGS, the Gaussian is defined in the plane. 
        # We assume p_l represents the 2D coordinates in the local frame.
        # To be precise, we should transform to canonical space for the Gaussian evaluation if needed,
        # but the paper uses G(p_i) directly in the sum. 
        # Let's follow the formula: G(x) = exp(-0.5 (x^T x)).
        # If p_l is the 2D coordinate in the local frame, we use it directly.
        g_val = math.exp(-0.5 * (p_l[0]**2 + p_l[1]**2))
        
        opacity_weight = prim.opacity * g_val
        contribution = transmittance * opacity_weight
        
        # 4. Compute color c_i
        # SH component
        sh_color = evaluate_sh(prim.sh_coeffs, ray.direction)
        
        # Texture component
        # u = p_l / k_i + T_offset
        k_i = prim.texel_size
        u = (p_l / k_i) + prim.t_offset
        
        # Bilinear interpolation of texture T_i at u
        # Handle boundary conditions (zero-padding if outside)
        texel_offset = bilinear_sample(prim.texture_map, u)
        
        c_i = sh_color + texel_offset
        
        # 5. Add to color
        color += contribution * c_i
        
        # 6. Update transmittance
        transmittance *= (1.0 - opacity_weight)
        
        if transmittance < 1e-6:
            break
            
    return color

def update_primitives(primitives, views, gt_images, iteration):
    """
    Updates primitive parameters and manages texture resolution.
    """
    # 1. Compute losses and gradients
    total_loss = 0.0
    per_primitive_errors = {}
    
    for view_idx, (view, gt_img) in enumerate(views):
        rendered_img = render_image(view, primitives)
        
        # L_RGB
        l1_loss = compute_l1(rendered_img, gt_img)
        ssim_loss = compute_ssim(rendered_img, gt_img)
        lambda_ssim = 0.2
        l_rgb = (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss
        
        # L_texture and L_opacity (global terms, but gradients flow to primitives)
        # L_texture = lambda_texture * sum(|cT_i|)
        # L_opacity = lambda_opacity * mean(opacity)
        
        # Compute per-primitive error E_i for densification
        for prim in primitives:
            # Get pixels covered by prim in this view
            covered_pixels = get_covered_pixels(prim, view)
            if not covered_pixels:
                continue
                
            # E_i^pi = sum(E_pi(r) * w_i^pi(r))
            # E_pi(r) is the pixel-wise error (e.g., L1)
            error_sum = 0.0
            weight_sum = 0.0
            for r in covered_pixels:
                pixel_error = abs(rendered_img[r] - gt_img[r]) # Simplified scalar error
                w = get_weight(prim, r) # Contribution of prim to pixel r
                error_sum += pixel_error * w
                weight_sum += w
            
            if weight_sum > 0:
                e_i_pi = error_sum / weight_sum
                per_primitive_errors[prim] = per_primitive_errors.get(prim, 0) + e_i_pi * weight_sum
                # Note: The formula for E_i uses w_i^pi as weight in sum, and denominator is sum of w_i^pi
                # We accumulate numerator and denominator separately
            else:
                per_primitive_errors[prim] = per_primitive_errors.get(prim, 0)
                
    # Normalize E_i
    for prim in primitives:
        if prim in per_primitive_errors:
            # Denominator is sum of weights over all views
            # We need to store weights sum separately or recompute
            # For simplicity, assume we tracked weight sums
            weight_sum_total = get_total_weight_sum(prim, views)
            if weight_sum_total > 0:
                prim.error = per_primitive_errors[prim] / weight_sum_total
            else:
                prim.error = 0.0
        else:
            prim.error = 0.0
            
    # 2. Apply Losses and Backprop
    # Compute L_texture and L_opacity
    l_texture = 0.0
    for prim in primitives:
        for texel in prim.texture_map:
            # cT_i = 2*sigmoid(c'_Ti) - 1
            # Loss is sum(|cT_i|)
            l_texture += abs(texel)
    l_texture *= lambda_texture
    
    l_opacity = 0.0
    for prim in primitives:
        l_opacity += prim.opacity
    l_opacity *= lambda_opacity / len(primitives)
    
    total_loss = l_rgb + l_texture + l_opacity
    total_loss.backward()
    
    # 3. Optimizer Step
    optimizer.step()
    
    # 4. Primitive Management (every 100 iterations)
    if iteration % 100 == 0:
        manage_primitives(primitives, views)

def manage_primitives(primitives, views):
    """
    Handles upscaling, downscaling, and splitting.
    """
    # Sort primitives by error
    sorted_prims = sorted(primitives, key=lambda p: p.error, reverse=True)
    
    # Top 10% for upscaling/splitting
    top_10_percent = sorted_prims[:len(sorted_prims)//10]
    
    for prim in top_10_percent:
        # Upscaling
        if prim.t2_pr >= 2:
            prim.t2_pr /= 2 # Halve t2_pr to upscale
            # Resize texture map to match new resolution
            resize_texture(prim.texture_map, prim.t2_pr)
        else:
            # Check for splitting (geometric error)
            if prim.texture_resolution > tau_tr:
                split_primitive(prim)

def split_primitive(prim):
    """
    Splits a primitive into two, displaced by +/- 1 std dev.
    """
    # Create two new primitives
    new_prims = []
    for sign in [-1, 1]:
        new_prim = copy(prim)
        
        # Displace position by +/- 1 std dev along axis
        # Assuming 2D Gaussian, we split along one axis (e.g., x)
        # Or split along both? Paper says "separately for each axis"
        # Let's assume we split along the axis with largest scale or just x
        # Paper: "displaced by +/- 1 standard deviation of the Gaussian along the axis"
        # "performed separately for each axis" implies we might do multiple splits?
        # "Replace the primitive with two new primitives" suggests one split at a time.
        
        # Let's split along the first axis (x)
        std_dev = prim.scale[0] # Scale is related to std dev
        displacement = sign * std_dev
        
        # Transform displacement to world space
        # p_new = mu + R * [displacement, 0]
        disp_local = [displacement, 0]
        disp_world = prim.R @ disp_local
        new_prim.mu = prim.mu + disp_world
        
        # Halve scale in corresponding axis
        new_prim.scale[0] = prim.scale[0] / 2
        
        # Half texture resolution
        new_prim.t2_pr = prim.t2_pr * 2 # Halving resolution means doubling t2_pr?
        # Wait, t2_pr is texel size to pixel size ratio.
        # If texture resolution is halved, the number of texels is halved.
        # If world size is same, texel size k must double.
        # k = k_min * t2_pr. So t2_pr doubles.
        
        # Opacity: G(1) times the opacity
        # G(1) = exp(-0.5)
        new_prim.opacity = prim.opacity * math.exp(-0.5)
        
        # Initialize texture by sampling original point
        new_prim.texture_map = sample_texture(prim.texture_map, new_prim)
        
        new_prims.append(new_prim)
        
    # Replace old primitive with new ones
    primitives.remove(prim)
    primitives.extend(new_prims)
```

## Operating characteristics

**Time and Memory Complexity:**
*   **Memory:** Each primitive has 59 parameters (4 bytes each) plus texture storage. Texture storage is jagged, with 3 floats per texel. Hard limit of 256 texture resolution per primitive [S1].
*   **Training Time:** 1.5-2x slower than 2DGS due to per-ray texture querying overhead [S1].
*   **Rendering Time:** 25% slower than 2DGS due to texture lookups [S1].

**Measured Performance:**
*   **DeepBlending:** SSIM 0.907, PSNR 30.03, LPIPS 0.303, 222K points, 21.6M texels, 78.1M params, 70 FPS [S1].
*   **Mip-Nerf-360:** SSIM 0.795, PSNR 27.00, LPIPS 0.263, 218K points, 46.6M texels, 152.6M params, 67 FPS [S1].
*   **Tanks&Temples:** SSIM 0.835, PSNR 23.43, LPIPS 0.225, 164K points, 10.4M texels, 41.1M params, 121 FPS [S1].

**Strengths and Limitations:**
*   **Strengths:** Content-aware dynamic texel allocation results in fewer total parameters than fixed-ratio methods (e.g., BBSplat). Avoids convergence issues of cloning.
*   **Limitations:** Slower training and rendering than 2DGS. Requires custom jagged tensor storage.

**When to Use:**
*   When high-frequency appearance details are needed without excessive parameter count.
*   When avoiding cloning-based densification is critical for stability.

**When Not to Use:**
*   When training speed is the primary constraint.
*   When hardware does not support efficient jagged tensor operations.

## Background

**Origin:**
The method was introduced by Panagiotis Papantonakis, Georgios Kopanas, Frédo Durand, and George Drettakis in the paper "Content-Aware Texturing for Gaussian Splatting" [S1].

**History and Development:**
It builds on 2D Gaussian Splatting (2DGS) and addresses limitations in texture mapping and densification. It differs from GSTex by optimizing primitives and textures from scratch rather than using a pretrained point cloud. It differs from BBSplat by using dynamic texel allocation instead of a fixed ratio [S1].

**Usage:**
Implemented using the 3DGS codebase, introducing 2D primitives as in 2DGS [S1].

## Sources

[S1] Content-Aware Texturing for Gaussian Splatting, https://doi.org/10.2312/sr.20251190, fetched 2026-10-26
